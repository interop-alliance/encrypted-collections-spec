# Encrypted Collections for Wallet Attached Storage

The WAS Encrypted Collections profile (WAS-EC): a client-side companion
profile to the [Wallet Attached Storage
spec](https://w3c-ccg.github.io/wallet-attached-storage-spec/) defining
end-to-end encrypted Collections -- the encryption descriptor with its
key-epoch roster, the Encrypted Data Vault envelope format with its
AEAD-bound `was` integrity binding, epoch-configuration authentication,
recipient management, and the deferred-minting rules for local-first
writers.

Referenced by the [App Connect
spec](https://interop-alliance.github.io/app-connect-spec/) as `[[WAS-EC]]`.

## Preview locally

```
npx serve .
```

Then open the served `index.html`; ReSpec renders client-side. The document
body lives in `spec.md` (Markdown, included by `index.html`).
