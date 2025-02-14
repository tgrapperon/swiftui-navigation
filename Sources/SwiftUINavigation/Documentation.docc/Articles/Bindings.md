# Bindings

Learn how to manage certain view state, such as `@FocusState` directly in your observable object.

## Overview

SwiftUI comes with many property wrappers that can be used in views to drive view state, such as 
`@FocusState`. Unfortunately, these property wrappers _must_ be used in views. It's not possible
to extract this logic to an observable object and integrate it with the rest of the model's
business logic, and be in a better position to test this state.

We can work around these limitations by introducing a published field to your observable
object and synchronizing it to view state with the `bind` view modifier that ships with this 
library.

For example, suppose you have a sign in flow where if the API request to sign in fails, you want
to refocus the email field. The model can be implement like so:

```swift
class SignInModel: ObservableObject {
  @Published var email: String
  @Published var password: String
  @Published var focus: Field?
  enum Field { case email, password }

  func signInButtonTapped() async {
    do {
      try await self.apiClient.signIn(self.email, self.password)
    } catch {
      self.focus = .email
    }
  }
}
```

Notice that we store the focus as a `@Published` property in the model rather than `@FocusState`.
This is because `@FocusState` only works when installed directly in a view. It cannot be used in
an observable object.

You can implement the view as you would normally, except you must also use `@FocusState` for the 
focus _and_ use the `bind` helper to make sure that changes to the model's focus are replayed to
the view, and vice versa.

```swift
struct SignInView: View {
  @FocusState var focus: SignInModel.Field?
  @ObservedObject var model: SignInModel

  var body: some View {
    Form {
      TextField("Email", text: self.$model.email)
      TextField("Password", text: self.$model.password)
      Button("Sign in") {
        Task {
          await self.model.signInButtonTapped()
        }
      }
    }
    // ⬇️ Replays changes of `model.focus` to `focus` and vice-versa.
    .bind(self.$model.focus, to: self.$focus)
  }
}
```
