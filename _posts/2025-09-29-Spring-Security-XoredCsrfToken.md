---
layout: post
title: "Contributing to Spring Security: Refactoring XOR-Based CSRF Token Encoding"
date: 2025-09-29 14:00:00 +0200
categories: Spring Security
---
## 🔒 Contributing to Spring Security: Refactoring XOR-Based CSRF Token Encoding

Recently, I had the opportunity to make a contribution to the [Spring Security](https://github.com/spring-projects/spring-security) project. In this post, I’d like to share the background, motivation, and technical details behind a refactoring I proposed and implemented — extracting the XOR-based CSRF token encoding logic into a dedicated, testable component.

---

### 🎯 The Problem: Hidden XOR Encoding Logic

Spring Security uses XOR-based encoding for CSRF tokens to prevent certain types of attacks. However, this encoding/decoding logic was previously implemented as **private static methods** within the following class:

```java
XorCsrfTokenRequestAttributeHandler
```

This design presented several limitations:

- 🔍 **Difficult to test independently**
    
- 🔒 **Logic was tightly coupled and hidden**
    
- 🔁 **Lacked reusability or flexibility**
    
- ❌ **Limited visibility into critical security logic**
    

---

### 🔍 The Original Code

Here is the original XOR-based encoding logic that was tightly embedded within the `XorCsrfTokenRequestAttributeHandler` class:

```java
private static String createXoredCsrfToken(SecureRandom secureRandom, String token) {
    byte[] tokenBytes = Utf8.encode(token);
    byte[] randomBytes = new byte[tokenBytes.length];
    secureRandom.nextBytes(randomBytes);

    byte[] xoredBytes = xorCsrf(randomBytes, tokenBytes);
    byte[] combinedBytes = new byte[tokenBytes.length + randomBytes.length];
    System.arraycopy(randomBytes, 0, combinedBytes, 0, randomBytes.length);
    System.arraycopy(xoredBytes, 0, combinedBytes, randomBytes.length, xoredBytes.length);

    return Base64.getUrlEncoder().encodeToString(combinedBytes);
}

private static byte[] xorCsrf(byte[] randomBytes, byte[] csrfBytes) {
    Assert.isTrue(randomBytes.length == csrfBytes.length, "arrays must be equal length");
    int len = csrfBytes.length;
    byte[] xoredCsrf = new byte[len];
    System.arraycopy(csrfBytes, 0, xoredCsrf, 0, len);
    for (int i = 0; i < len; i++) {
        xoredCsrf[i] ^= randomBytes[i];
    }
    return xoredCsrf;
}
```

As you can see, this code is:

- Defined as **private static methods**, making it impossible to test directly
    
- Embedded within a handler class that has a different primary responsibility, reducing modularity and violating the Single Responsibility Principle (SRP)
    
- Not reusable or extensible outside of this specific context
    

This raised concerns, especially for a security-critical feature like CSRF protection.

---

### 🛠️ The Proposal: Make XOR Encoding a First-Class Citizen

To address these limitations, I opened [Issue #17968](https://github.com/spring-projects/spring-security/issues/17968) proposing the following:

> **Expected Behavior**  
> Refactor the XOR-based CSRF token encoding and decoding logic into a dedicated, public class (e.g., `XorCsrfTokenEncoder`) with publicly accessible methods to allow unit testing, improve maintainability, and support future extensions.

---

### ✅ Pull Request Overview

You can find the PR here: [#17969](https://github.com/spring-projects/spring-security/pull/17969)

Here’s a summary of the key changes I made:

- ✅ **Introduced a new `XorCsrfTokenEncoder` class**
    
    - Public `encode()` and `decode()` methods
        
    - Focused solely on XOR logic
        
- ✅ **Introduced a `CsrfTokenEncoder` interface**
    
    - Defines a contract for encoding/decoding
        
    - Lays groundwork for future extensibility
        
- ✅ **Updated `XorCsrfTokenRequestAttributeHandler`**
    
    - Delegates XOR logic to the new encoder
        
    - Supports injecting custom `SecureRandom`
        
- ✅ **Added comprehensive unit tests**
    
    - Increased test coverage and reliability
        

---

### 🧱 Refactored Code Structure

#### `CsrfTokenEncoder` Interface

```java
public interface CsrfTokenEncoder {

	String encode(String token);

	@Nullable
	String decode(String encodedToken, String originalToken);

}
```

This interface defines a contract for token encoding and decoding. While only one implementation (`XorCsrfTokenEncoder`) currently exists, this abstraction prepares the system for future extensibility (e.g., token signing or encryption).

#### `XorCsrfTokenRequestAttributeHandler` (Updated)

```java
public final class XorCsrfTokenRequestAttributeHandler extends CsrfTokenRequestAttributeHandler {

	private CsrfTokenEncoder csrfTokenEncoder = new XorCsrfTokenEncoder();

	public void setSecureRandom(SecureRandom secureRandom) {
		Assert.notNull(secureRandom, "secureRandom cannot be null");
		this.csrfTokenEncoder = new XorCsrfTokenEncoder(secureRandom);
	}

	@Override
	public void handle(HttpServletRequest request, HttpServletResponse response,
			Supplier<CsrfToken> deferredCsrfToken) {
		// ...
		Supplier<CsrfToken> updatedCsrfToken = deferCsrfTokenUpdate(deferredCsrfToken);
		super.handle(request, response, updatedCsrfToken);
	}

	private Supplier<CsrfToken> deferCsrfTokenUpdate(Supplier<CsrfToken> csrfTokenSupplier) {
		return new CachedCsrfTokenSupplier(() -> {
			CsrfToken csrfToken = csrfTokenSupplier.get();
			String updatedToken = csrfTokenEncoder.encode(csrfToken.getToken());
			return new DefaultCsrfToken(csrfToken.getHeaderName(), csrfToken.getParameterName(), updatedToken);
		});
	}

	@Override
	public @Nullable String resolveCsrfTokenValue(HttpServletRequest request, CsrfToken csrfToken) {
		String actualToken = super.resolveCsrfTokenValue(request, csrfToken);
		if (actualToken == null) {
			return null;
		}
		return csrfTokenEncoder.decode(actualToken, csrfToken.getToken());
	}
}
```

The updated handler now delegates encoding responsibilities to the `CsrfTokenEncoder`, following SRP and significantly improving testability.

---

### 🔄 Before vs After

|Before|After|
|---|---|
|XOR logic hidden in private static methods|Public, reusable, testable class|
|No direct unit tests possible|Full test coverage of XOR logic|
|Tight coupling|Decoupled, interface-driven design|
|No room for extension|Prepared for future extensibility|

---

### 💡 Why This Matters

Although this PR doesn't introduce new functionality, it makes important architectural improvements:

- Enhances **modularity and testability**
    
- Improves **code clarity and maintainability**
    
- Enables developers to write **focused unit tests** for this critical part of the framework
    
- Lays the foundation for **future extensibility**, without enforcing it prematurely
    

Security features — especially token encoding — should be easy to inspect, test, and evolve. This refactor helps achieve exactly that.

---

### 🚀 Looking Ahead

While the current design is still tightly focused on XOR-based CSRF token handling, the introduction of the `CsrfTokenEncoder` interface opens the door for future enhancements, such as:

- Alternative encoding strategies
    
- Custom token formats
    
- Token signing or encryption, if needed
    

That said, broader application of the **Open/Closed Principle** would require additional architectural changes to support pluggable encoders at a higher level.

---

### 📝 Final Thoughts

This contribution reminded me that **small, focused refactors** can have a big impact on a codebase — especially when they touch core security logic. I’m grateful for the opportunity to contribute to such a widely-used and respected project like Spring Security.

If you're interested, you can check out the full PR here:

👉 Pull Request: [#17969](https://github.com/spring-projects/spring-security/pull/17969)
👉 Related Issue: [#17968](https://github.com/spring-projects/spring-security/issues/17968)

Thanks for reading!


---
