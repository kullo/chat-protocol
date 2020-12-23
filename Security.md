Security considerations
=======================

## The server can actively attack the user by

1. sending arbitrary authenticated requests after copying the authentiication data `signed(LoginKey)`,
2. pretending to know the plain password by coping the unhashed `PasswordVerificationKey`,
3. verify pending user registrations that are sent from the attacker,

* Using 2. the server can add a new device to an existing user and send signed messages.
