---
layout: single
title: "Azure APIM Global Error Handling Gotchas"
date: 2025-07-20 10:00:00 -0000
categories:
  - azure
  - development
tags:
  - azure
  - apim
  - api-management
  - policies
  - error-handling
author: Ryan Gammon
toc: true
toc_sticky: true
---

_Or: How I learned to stop worrying and love the policy hierarchy_

## TL;DR

If you're getting errors like "Element is not allowed in global context" when trying to use `<base />` in your Azure APIM global policy's `<on-error>` section, you're not alone. **Global policies can't use `<base />` in error handling because there's nothing above them to inherit from.** This creates some interesting challenges when you want both default error handling AND custom global error headers like correlation IDs.

## The Problem: When Your Global Error Policy Breaks Everything

Picture this: You've got a beautiful Azure API Management setup with multiple APIs. You want to add a correlation ID to all error responses for better traceability. Naturally, you think "I'll just add this to my global policy's `<on-error>` section!"

So you write something like this:

```xml
<policies xmlns:xi="http://www.w3.org/2001/XInclude">
    <inbound>
        <set-variable name="correlationId" value="@(Guid.NewGuid().ToString())" />
        <!-- other global stuff -->
    </inbound>

    <outbound>
        <!-- response headers -->
    </outbound>

    <on-error>
        <base /> <!-- 💥 BOOM! This breaks everything -->
        <set-header name="X-Correlation-ID" exists-action="override">
            <value>@((string)context.Variables["correlationId"])</value>
        </set-header>
    </on-error>
</policies>
```

And then Azure slaps you with:

```
One or more fields contain incorrect values:
Error in element 'base' on line X, column Y: Element is not allowed in global context
```

## Understanding the APIM Policy Hierarchy

The issue here is that **APIM policies work in a hierarchy**, and `<base />` means "call the parent policy." But global policies don't have parents - they're at the top of the food chain!

Here's how the hierarchy works:

```
Global Policy (no parent - can't use <base />)
└── API Policy (can use <base /> to call Global)
    └── Operation Policy (can use <base /> to call API)
```

According to [Microsoft's policy documentation](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-policies), the `<base />` element "applies the policies defined at a higher scope to the current scope." Since global policies are the highest scope, there's literally nothing higher to call.

## The Catch-22: You Want Both Default Behavior AND Custom Headers

This creates a frustrating situation:

- **Without `<base />`**: You lose APIM's built-in error handling and have to manually handle every error scenario
- **With custom `<on-error>`**: You override the default behavior completely
- **Without any `<on-error>`**: You can't add your correlation ID to error responses

It's like being told you can have either chocolate or vanilla, but not chocolate-vanilla swirl.

## Solution 1: Skip Global Error Handling (Recommended)

**The best approach?** Don't use global `<on-error>` at all. Let each API handle its own errors and add your correlation ID in the global `<outbound>` section instead:

```xml
<policies xmlns:xi="http://www.w3.org/2001/XInclude">
    <inbound>
        <set-variable name="correlationId" value="@(Guid.NewGuid().ToString())" />
        <set-header name="X-Correlation-ID" exists-action="override">
            <value>@((string)context.Variables["correlationId"])</value>
        </set-header>
        <!-- other global stuff -->
    </inbound>

    <outbound>
        <!-- This runs for ALL responses, including errors! -->
        <set-header name="X-Correlation-ID" exists-action="override">
            <value>@((string)context.Variables["correlationId"])</value>
        </set-header>
        <!-- other response headers -->
    </outbound>

    <!-- No <on-error> section - let individual APIs handle errors -->
</policies>
```

**Why this works:**

- The `<outbound>` section runs for **all responses**, including error responses
- Individual APIs can use `<base />` in their `<on-error>` sections (since they have a parent)
- You get consistent correlation IDs without breaking default error handling

## Solution 2: Add Error Handling to Individual APIs

If you need more control, add error handling at the API level where `<base />` actually works:

```xml
<!-- In your API-level policy -->
<policies>
    <inbound>
        <base /> <!-- Calls global policy -->
        <!-- API-specific stuff -->
    </inbound>

    <outbound>
        <base /> <!-- Calls global policy -->
        <!-- API-specific stuff -->
    </outbound>

    <on-error>
        <base /> <!-- This works! Calls global policy -->
        <!-- API-specific error handling -->
        <set-header name="X-API-Error-Source" exists-action="override">
            <value>Recipe API</value>
        </set-header>
    </on-error>
</policies>
```

## Solution 3: Go Full Custom (Not Recommended)

You _could_ implement all error handling manually in your global policy, but this means recreating all of APIM's built-in error handling:

```xml
<on-error>
    <set-header name="X-Correlation-ID" exists-action="override">
        <value>@((string)context.Variables["correlationId"])</value>
    </set-header>

    <!-- Now you have to handle every possible error scenario -->
    <choose>
        <when condition="@(context.Response.StatusCode == 404)">
            <!-- Keep 404 as-is -->
        </when>
        <when condition="@(context.Response.StatusCode == 401)">
            <!-- Keep 401 as-is -->
        </when>
        <otherwise>
            <!-- Custom handling for everything else -->
            <set-status code="500" reason="Internal Server Error" />
            <set-body>{"error": "Something went wrong"}</set-body>
        </otherwise>
    </choose>
</on-error>
```

This is a maintenance nightmare and you'll miss edge cases that APIM handles automatically.

## What We Learned the Hard Way

During our debugging session, we discovered that:

1. **Global `<on-error>` blocks can interfere with API-specific error handling** - even when they seem fine, they can cause unexpected 500 errors

2. **The error message about `<base />` not being allowed is actually helpful** - it's telling you about the hierarchy, not just being difficult

3. **APIM's default error handling is pretty good** - it properly returns 404s for missing resources, handles authentication errors, etc.

4. **Correlation IDs work great from the `<outbound>` section** - no need to overcomplicate things

## The Bottom Line

**Don't fight the hierarchy.** APIM's policy system is designed with a clear inheritance model, and trying to work against it usually leads to frustration. The global `<outbound>` section is your friend for adding headers to all responses, including errors.

If you absolutely need custom error handling, do it at the API or operation level where `<base />` actually works. Your future self (and your teammates) will thank you for keeping things simple and predictable.

## References

- [Azure API Management Policies Overview](https://docs.microsoft.com/en-us/azure/api-management/api-management-howto-policies)
- [Policy Expressions in Azure API Management](https://docs.microsoft.com/en-us/azure/api-management/api-management-policy-expressions)
- [Advanced API Management Policies](https://docs.microsoft.com/en-us/azure/api-management/api-management-advanced-policies)

---

_Have you run into similar APIM policy gotchas? Feel free to open an issue on the [blog's GitHub repo](https://github.com/rggammon/gamnation-blog/issues) to share your war stories! 😅_
