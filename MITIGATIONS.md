# Mitigation and Remediation Guide

## WebView Labs

VULN-WV-01: Only enable JavaScript for trusted content. Use shouldOverrideUrlLoading allowlist.

VULN-WV-02: Avoid addJavascriptInterface. Restrict to trusted origins, minimize exposed methods.

VULN-WV-03: Set setAllowFileAccess false, setAllowFileAccessFromFileURLs false, setAllowUniversalAccessFromFileURLs false.

VULN-WV-04: Validate intent-supplied URLs against an allowlist before loadUrl.

## Deeplink Labs

VULN-DL-01 and DL-02: Never use Class.forName with attacker-controlled strings. Use a hardcoded allowlist of destinations.

VULN-DL-03: Deeplinks must never modify auth state without a signed, validated token.

VULN-DL-CHAIN: Fix WV-02 and DL-02 independently. Defense in depth.

## SQL Injection

VULN-SQL-01: Use parameterized queries with rawQuery and a parameters array instead of string concatenation. Store password hashes using bcrypt or Argon2, never plaintext.

## Admin Panel Labs

VULN-AP-01: Set exported false unless external launch required. Use signature permissions.

VULN-AP-02: Never hardcode secrets. Retrieve at runtime after authentication.

VULN-AP-03: Verify active server-side session before rendering privileged UI.

## Broadcast Receiver Labs

VULN-BR-01 and BR-02: Set exported false. Require signature permission for cross-app broadcasts.

VULN-BR-03: Never log tokens or secrets. Strip debug logs from release builds.

## Service Labs

VULN-SV-01, SV-02a, SV-02b: Set exported false. Validate caller signature. Allowlist permitted actions. Never accept attacker-supplied exfil URLs.

## General Hardening Checklist

- Set debuggable false in release builds
- Avoid usesCleartextTraffic true
- Enable certificate pinning
- Run apktool and jadx against your own release APK to verify no secrets leak
- Use lint and MobSF in CI

## LFI Labs

VULN-LFI-01 - Local File Inclusion via path traversal

Fix: Never concatenate user-supplied filenames into file paths. Use Path.normalize and verify the resolved path stays within the intended directory (canonical path prefix check). Alternatively, use an indirect reference (an opaque ID mapped server-side to a real filename) instead of accepting a filename directly.

### How LFI Can Escalate to RCE (Conceptual)

In this lab, LFI-01 only reads a text file within the app sandbox - it does not execute anything, so it cannot lead to code execution as written.

However, in real-world apps, LFI becomes RCE when the file being read is later treated as executable content rather than inert data. Common patterns researchers should watch for:

1. WebView + LFI: if a path-traversal-readable file is rendered inside a WebView via loadUrl("file://..."), and that file contains HTML/JS, the traversed file's script executes in the WebView's JS context (chains with VULN-WV-02/03 in this lab - this is exactly what VULN-DL-CHAIN demonstrates conceptually).

2. Dynamic code loading: if an app uses DexClassLoader or PathClassLoader to load a .dex/.jar file by a path derived from user input, an LFI-style traversal that points at an attacker-writable location (e.g., external storage, a cache directory populated by another vulnerable component) can result in arbitrary code execution when that file is loaded and classes are instantiated.

3. Native library loading: System.load() or System.loadLibrary() with a path influenced by traversal can load an attacker-supplied .so file, executing native code in the app's process.

4. Config/template injection: if the traversed file is a configuration or template file that the app later parses and acts on (e.g., a script runner, a deserialization target, or a templating engine), attacker-controlled content in that file can result in code execution during parsing.

Mitigation for all of the above: never combine "attacker can influence which file is read" with "the result of that read is treated as executable code, a class, a library, or a deserialization target." Treat file content read via any user-influenced path as untrusted data only.
