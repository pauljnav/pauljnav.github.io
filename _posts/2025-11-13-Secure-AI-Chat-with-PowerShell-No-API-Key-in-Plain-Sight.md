# Secure AI Chat with PowerShell: No API Key in Plain Sight 🔑

I've just published a new PowerShell function, **`Send-AIChat.ps1`**, designed for easily interacting with AI APIs. 
While the function itself simplifies the API call, its most critical feature I'm demonstrating within, is how it 
handles your sensitive **API key**.

### The Problem with Plain Strings

Using a plaintext string for your API key can be a security risk.

Having a plaintext string for your API key in your scripts **is** a security risk.

### The Correct Local Way: SecureString and BSTR

To protect this key, the function relies on the `.NET` framework's **`SecureString`** type combined with the **BSTR** marshaling method.

1.  We store the key as a **`SecureString`** to keep it encrypted in memory.
2.  Before sending the request, we use `[System.Runtime.InteropServices.Marshal]::SecureStringToBSTR()` to temporarily decrypt the key into **unmanaged memory**.
3.  For enhancing security, we then use **`[System.Runtime.InteropServices.Marshal]::ZeroFreeBSTR()`** to **explicitly wipe and free** that unmanaged memory immediately after the key is used.

This is a standard and secure pattern for handling secrets that must be briefly converted to plaintext for external consumption (like an API header).

Check out the function and its secure implementation from its location in my public [PSfunctions](https://github.com/pauljnav/PSfunctions) repo:
[Send-AIChat.ps1 on GitHub](https://github.com/pauljnav/PSfunctions/blob/main/Send-AIChat.ps1)
