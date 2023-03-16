---
title: "Implementing a safe and sound API Key authorization middleware in Go"
datePublished: Sun Feb 06 2022 02:40:00 GMT+0000 (Coordinated Universal Time)
cuid: clf8ylnde079hxnnv69lv3nyz
slug: implementing-a-safe-and-sound-api-key-authorization-middleware-in-go
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1678882731600/b5fc8455-409e-4439-9cad-e79806848551.jpeg
tags: go, security, authorization, hashing

---

---

A common requirement that I face on multiple projects is to safeguard some API endpoints to administrative access, or to provide a secure way for other applications to consume our service in a controlled and traceable manner.

The usual solution for it is API Keys, a simple and effective authorization control mechanism that we can implement with a few lines of code. However, when doing, so we also need to be aware of threats and possible attacks that we may suffer, specially due to the usual privileges that these keys provides.

Therefore, we are going to analyze common points of concern and design a solution that improve our security posture while keeping it simple.

## API Keys threats

There are two main concerns when implementing an API Key authorization scheme: **key provisioning** and **timing attacks**. Let’s review each threat before designing solutions to address them.

### Key Provisioning

The key storage is directly related to how applications expect these secrets to be provided to them. Environment variables are the most common solution used on modern services since they are widely supported and don’t incur a high reading cost (in contrast to files) allowing for dynamic changes to be easily detected.

However, the way developers usually define the environment variables are through scripts or configuration files, for example using a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) manifest. This introduces a serious threat of API Keys being committed to git repositories, which in the event of data leakage from the internal VCS management system would expose these credentials.

Note: remember that once committed, even if the keys are deleted from the source files, the information is already on the repository history and is easily searchable with tools like [TruffleHog](https://github.com/trufflesecurity/truffleHog).

Therefore, **please do not commit your API Keys to git**!

### Timing Attacks

Once your application is configured with the available API Keys, you need to verify that the end-user provided key (let’s call this the *user key*) is correct. Doing so with a naive algorithm, like using == operator, will make the verification end on the first incorrect character, hence reducing the time taken to respond.

A timing attack takes advantage of this scenario by trying to guess the correct characters of a secret based on how long the application took to respond. If the guess is right, the response will take slightly longer than if it’s wrong.

Naturally, since equality checks are orders of magnitude faster than the network roundtrip, this type of attack is extremely difficult to perform because it depends on a statistical analysis of many response samples. By looking at the time distribution produced by two different characters, one can infer that if they are different, inferring that the greater one is the correct value. For an extensive discussion of statistical techniques to help perform this attack see [Morgan, Morgan 2015](https://www.blackhat.com/docs/us-15/materials/us-15-Morgan-Web-Timing-Attacks-Made-Practical-wp.pdf).

## Middleware design and implementation

Having these threats in mind, we can design a suitable solution. Let’s start with the most simple API Key middleware implementation possible and iterate from it.

```go
func ApiKeyMiddleware(cfg conf.Config, logger logging.Logger) func(handler http.Handler) http.Handler {
	apiKeyHeader := cfg.APIKeyHeader // string
	apiKeys := cfg.APIKeys // map[string]string

	reverseKeyIndex := make(map[string]string)
	for name, key := apiKeys {
		reverseKeyIndex[key] = name
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			apiKey, err := bearerToken(r, apiKeyHeader)
			if err != nil {
				logger.Errorw("request failed API key authentication", "error", err)
				RespondError(w, http.StatusUnauthorized, "invalid API key")
				return
			}

			_, found := reverseKeyIndex[apiKey]
			if !found {
				hostIP, _, err := net.SplitHostPort(r.RemoteAddr)
				if err != nil {
					logger.Errorw("failed to parse remote address", "error", err)
					hostIP = r.RemoteAddr
				}
				logger.Errorw("no matching API key found", "remoteIP", hostIP)

				RespondError(w, http.StatusUnauthorized, "invalid api key")
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// bearerToken extracts the content from the header, striping the Bearer prefix
func bearerToken(r *http.Request, header string) (string, error) {
	rawToken := r.Header.Get(header)
	pieces := strings.SplitN(rawToken, " ", 2)

	if len(pieces) < 2 {
		return "", errors.New("token with incorrect bearer format")
	}

	token := strings.TrimSpace(pieces[1])

	return token, nil
}
```

A middleware is a function that takes an `http.Handler` and returns an `http.Handler`. In this code, the function `ApiKeyMiddleware` is a factory that creates an instance of the middleware with the provided configuration and logger. The `config.Config` is a struct populated from environment variables and `logging.Logger` is an interface that can be implemented using any logging library or the standard library. You could pass only the header and map of keys, but for clarity we choose to denote the dependency from this middleware to the configuration.

After extracting the fields that it relies on, the function creates a reverse index of the API Keys, which is originally a map from a key id/name to the key value. Using this reverse index it’s trivial to verify if the user key is valid by doing a map lookup on line 18.

However, this approach expects the API Keys as plaintext values and is susceptible to timing attacks, because its validation algorithm is not constant time.

### Using key hashes for validation

To improve the key provisioning workflow, we can use a simple yet effective solution: expect the available keys to be hashes. Using this approach we can now commit our key hashes to our repository because even in the event of a data leak they could not be reversed to their original value.

Let’s use the SHA256 hashing algorithm to encode our keys. For example, if one of them is `123456789` (please, do not use a key like this :D) then its hash will be:

```plaintext
15e2b0d3c33891ebb0f1ef609ec419420c20e320ce94c65fbc8c3312448eb225
```

Now you can add this hash to your deployment script, Kubernetes Secret, etc., and commit it with peace of mind.

Next, we need to handle this new format on our middleware. This is what the code will look like now:

```go
func ApiKeyMiddleware(cfg conf.Config, logger logging.Logger) func(handler http.Handler) http.Handler {
	apiKeyHeader := cfg.APIKeyHeader // string
	apiKeys := cfg.APIKeys // map[string]string

	reverseKeyIndex := make(map[string]string)
	for name, key := apiKeys {
		reverseKeyIndex[key] = name
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			apiKey, err := bearerToken(r, apiKeyHeader)
			if err != nil {
				logger.Errorw("request failed API key authentication", "error", err)
				RespondError(w, http.StatusUnauthorized, "invalid API key")
				return
			}

			_, ok := apiKeyIsValid(apiKey, reverseKeyIndex)
			if !ok {
				hostIP, _, err := net.SplitHostPort(r.RemoteAddr)
				if err != nil {
					logger.Errorw("failed to parse remote address", "error", err)
					hostIP = r.RemoteAddr
				}
				logger.Errorw("no matching API key found", "remoteIP", hostIP)

				RespondError(w, http.StatusUnauthorized, "invalid api key")
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

// apiKeyIsValid checks if the given API key is valid and returns the principal if it is.
func apiKeyIsValid(rawKey string, availableKeys map[string][]byte) (string, bool) {
	hash := sha256.Sum256([]byte(rawKey))
	key := string(hash[:])

	name, found := reverseKeyIndex[apiKey]

	return name, found
}

// bearerToken function omitted..
```

Here we extracted the logic to validate the key into a function that, before checking the equality of the user key against the available ones, encodes the user key using the same SHA256 algorithm.

This simple step improved a lot our security posture without adding much complexity. Now we can have the benefits of version control, like change history and easy detection when someone changes a key hash.

This approach works well when there are few keys to be managed, and you want to follow a GitOps approach. However, if you need to scale the key management, allow for self-service key requests and automatic rotation, you may want to look for a solution like [Hashicorp Vault](https://www.vaultproject.io). Even using an external secret store I still believe this strategy, to rely on key hashes to be valid, because your external secret store can persist both the original key and the hash, and the access policy for the application can have fewer privileges in such a way that it can only read the hashes.

### Constant time key verification

Once we have a better strategy to provision our keys, we need to defend ourselves against them being exfiltrated by timing attacks. The solution for this kind of vulnerability is to use an algorithm that takes the same time to produce a result whether the keys are equal or not. This is called a constant time comparison, and the Go Standard Library offers us an implementation in the `crypto/subtle` package that is perfect to solve most of our problems. Hence, we can update our code to use this package:

```go
func ApiKeyMiddleware(cfg conf.Config, logger logging.Logger) (func(handler http.Handler) http.Handler, error) {
	apiKeyHeader := cfg.APIKeyHeader
	apiKeys := cfg.APIKeys
	apiKeyMaxLen := cfg.APIKeyMaxLen

	decodedAPIKeys := make(map[string][]byte)
	for name, value := range apiKeys {
		decodedKey, err := hex.DecodeString(value)
		if err != nil {
			return nil, err
		}

		decodedAPIKeys[name] = decodedKey
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			ctx := r.Context()

			apiKey, err := bearerToken(r, apiKeyHeader)
			if err != nil {
				logger.Errorw("request failed API key authentication", "error", err)
				RespondError(w, http.StatusUnauthorized, "invalid API key")
				return
			}

			if _, ok := apiKeyIsValid(apiKey, decodedAPIKeys); !ok {
				 hostIP, _, err := net.SplitHostPort(r.RemoteAddr)
					if err != nil {
						logger.Errorw("failed to parse remote address", "error", err)
						hostIP = r.RemoteAddr
					}
					logger.Errorw("no matching API key found", "remoteIP", hostIP)

					RespondError(w, http.StatusUnauthorized, "invalid api key")
					return
			}

			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}, nil
}

// apiKeyIsValid checks if the given API key is valid and returns the principal if it is.
func apiKeyIsValid(rawKey string, availableKeys map[string][]byte) (string, bool) {
	hash := sha256.Sum256([]byte(rawKey))
	key := hash[:]

	for name, value := range availableKeys {
		contentEqual := subtle.ConstantTimeCompare(value, key) == 1

		if contentEqual {
			return name, true
		}
	}

	return "", false
}

// bearerToken function omitted...
```

Now, the function `apiKeyIsValid` uses `subtle.ConstantTimeCompare` to verify the user key against each available key. Since `subtle.ConstantTimeCompare` operates upon byte slices we don’t cast our hash to string anymore and also our reversed index has gone in place of a decoded map.

The decoding is necessary because the string representation of our key hashes are actually a hexadecimal encoding of the binary value. Hence, we cannot just cast the string to byte slice because Go assumes all strings to be UTF-8 encoded.

> Note: for an example on how using a cast instead of the correct decoding function, the result of `[]byte("09")` is `110000111001` while `hex.DecodeString("09")` produces `1001`. Check out the live example [here](https://go.dev/play/p/CPy16o7hvDO).

The major disadvantage of this solution is that now we need to iterate over all available keys before finding out if the key is incorrect. This doesn’t scale well if there are too many keys, however one simple workaround would be to require the client to send an extra header with the key ID/name, e.g. `X-App-Key-ID`, with which you can find the key in `O(1)` and then apply the constant time comparison.

However, there is one subtle (*pun intended*) behavior from `subtle.ConstantTimeCompare` that we must be aware before deploying our solution to production. When the byte slices have different lengths, the functions returns earlier without performing the bitwise operations. This is natural because it does an XOR between each pair of bits from each slice, and with slices of different sizes, there would be bits from one slice without a matching pair to be combined with. **Because of it, an adversary could measure that keys with the wrong length have a smaller response time than keys with the correct length, hence leaking the key length**. It would only be a vulnerability if you use a short key that is easily brute-forced, but with a simple 30 character key using the UTF-8 printable characters you would have `30^95 = 2.12089515 × 10^140` possible keys.

Finally, we’ve built a simple, secure and efficient API Key solution that should handle a lot of uses cases without additional infrastructure or complexity. Using a basic understanding of threats and the Golang standard library, we could do a security-oriented design instead of leaving security as an after-though in an iterative way.

Photo by [Silas Köhler](%5Bhttps://unsplash.com/@silas%5D(https://unsplash.com/@silas)_crioco?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](%5Bhttps://unsplash.com/s/photos/key?utm%5D(https://unsplash.com/s/photos/key?utm)_source=unsplash&utm_medium=referral&utm_content=creditCopyText)