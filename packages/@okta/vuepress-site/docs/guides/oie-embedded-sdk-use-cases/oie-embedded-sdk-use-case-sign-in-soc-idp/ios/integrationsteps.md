## Integration steps

### Summary

There are many different ways to integrate the Swift SDK into your app. Using
the sample code as a wrapper around the SDK, this guide describes the [steps](#steps)
involved in integrating this use case into your application. Feel free to modify
the sample code to fit your individual needs. The diagram below illustrates
how the sample code and SDK fit into the overall integration of your app.

<div class="common-image-format">

 ![Diagram showing the integration flow of the sample app and Swift SDK](/img/oie-embedded-sdk/oie-embedded-sdk-swift-sample-code-overview.png
 "Swift sample and sdk diagram")

</div>

### Steps

#### Step 1: Initialize SocialLogin

The first step is to initialize `SocialLogin` and pass in a
`configuration` object.

 ```swift
self.authHandler = SocialLogin(configuration: configuration)
```

For more information on how to set the `configuration` parameter, see
[SDK](/docs/guides/oie-embedded-common-download-setup-app/ios/main/#sdk)
in
[Download and set up the SDK, Sign-In Widget, and sample app](/docs/guides/oie-embedded-common-download-setup-app/ios/main)


#### Step 2: Call the login method

The next step is to call the `login` method passing in the username
and password. The method can take a completion handler which
will be invoked once, either with a fatal error, or success
with a token.

Example

```swift
self.authHandler?.login(service: .facebook)
 { result in
     switch result {
     case .success(let token):
         print(token)
     case .failure(let error):
         print(error)
     }
}
```

#### Step 3: Get user profile information (optional)

Optionally, you can obtain basic user information after a successful social
sign in by making a request to Okta's Open ID Connect authorization server. See
[Get user profile information after sign in](/docs/guides/oie-embedded-sdk-alternate-flows/ios/main/#get-user-profile-information-after-sign-in).

## Sample code

```swift
public class SocialLogin {
    private let configuration: IDXClient.Configuration
    private weak var presentationContext: ASWebAuthenticationPresentationContextProviding?
    private var webAuthSession: ASWebAuthenticationSession?

    private var client: IDXClient?
    private var completion: ((Result<IDXClient.Token, LoginError>) -> Void)?

    public init(configuration: IDXClient.Configuration) {
        self.configuration = configuration
    }

    public func login(service: IDXClient.Remediation.SocialAuth.Service, from presentationContext: ASWebAuthenticationPresentationContextProviding? = nil, completion: @escaping (Result<IDXClient.Token, LoginError>) -> Void) {
        self.presentationContext = presentationContext
        self.completion = completion

        IDXClient.start(with: configuration) { (client, error) in
            guard let client = client else {
                self.finish(with: error)
                return
            }

            self.client = client
            client.resume { (response, error) in
                guard let response = response else {
                    self.finish(with: error)
                    return
                }

                guard let remediation = response.remediations.first(where: { remediation in
                    let socialRemediation = remediation as? IDXClient.Remediation.SocialAuth
                    return socialRemediation?.service == service
                }) as? IDXClient.Remediation.SocialAuth
                else {
                    self.finish(with: .cannotProceed)
                    return
                }

                DispatchQueue.main.async {
                    self.login(with: remediation)
                }
            }
        }
    }

    func login(with remediation: IDXClient.Remediation.SocialAuth) {
        guard let client = client,
              let scheme = URL(string: client.context.configuration.redirectUri)?.scheme
        else {
            finish(with: .cannotProceed)
            return
        }

        let session = ASWebAuthenticationSession(url: remediation.redirectUrl,
                                                 callbackURLScheme: scheme)
        { [weak self] (callbackURL, error) in
            guard error == nil,
                  let callbackURL = callbackURL,
                  let client = self?.client
            else {
                self?.finish(with: error)
                return
            }

            let result = client.redirectResult(for: callbackURL)

            switch result {
            case .authenticated:
                client.exchangeCode(redirect: callbackURL) { (token, error) in
                    guard let token = token else {
                        self?.finish(with: error)
                        return
                    }
                    self?.finish(with: token)
                }

            default:
                self?.finish(with: .cannotProceed)
            }
        }

        session.presentationContextProvider = presentationContext
        session.prefersEphemeralWebBrowserSession = true
        session.start()

        self.webAuthSession = session
    }

    public enum LoginError: Error {
        case error(_ error: Error)
        case message(_ string: String)
        case cannotProceed
        case unknown
    }
}

// Utility functions to help return responses to the caller.
extension SocialLogin {
    func finish(with error: Error?) {
        if let error = error {
            finish(with: .error(error))
        } else {
            finish(with: .unknown)
        }
    }

    func finish(with error: LoginError) {
        completion?(.failure(error))
        completion = nil
    }

    func finish(with token: IDXClient.Token) {
        completion?(.success(token))
        completion = nil
    }
}
```