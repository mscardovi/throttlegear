# ThrottleGear XML Cryptography & Tool Internals

This document details the cryptography parameters, XML structure, mapping specifications, and the internal design of the Rust-based ASUS ThrottleGear tool.

---

## Part 1: XML Specifications & Cryptography Design

This section covers how the ASUS ThrottleGear XML configurations are structured, located, encrypted, and mapped.

### 1.1 Default File Locations (Windows)
On Windows, the ASUS Armoury Crate services store the ThrottleGear XML configuration files in the following system folders (inside the hidden `C:\ProgramData` folder by default):

#### A. Default Device Configurations
Contains the baseline configurations for different device/model profiles (e.g., `ThrottleGear_YOURMODEL.xml`):
```
C:\ProgramData\ASUS\Armoury Crate Config\Data\
```

#### B. Manual / DC Manual User Profiles
Contains user-customized manual mode and DC manual mode profiles (e.g., `ThrottleGearManual.xml`):
```
C:\ProgramData\ASUS\Armoury Crate Service\Data\
```

---

### 1.2 Cryptography Parameter Derivation
The encryption uses **AES-256-CBC** with PKCS7 padding. The cryptographic key and Initialization Vector (IV) are derived directly from the root attributes of the `<ThrottlePluginConfig>` XML element:

#### A. Key Derivation (32 Bytes / 256 Bits)
The key is determined by the `Type` and `ModelName` attributes:
1. A 32-byte array initialized to `0`.
2. The first byte (index `0`) is set to:
   - `1` if `Type` is `"DT"` (Desktop).
   - `0` otherwise (e.g., `"NB"` for notebooks, `"NUC"`).
3. The ASCII bytes of the `ModelName` attribute are copied into the key array starting at index `1`, up to a maximum of 31 bytes.
4. The remaining bytes of the array are left as `0`.

**C# Reference:**
```csharp
byte[] key = new byte[32];
key[0] = (type == "DT") ? (byte)1 : (byte)0;
Array.Copy(Encoding.ASCII.GetBytes(modelName), 0, key, 1, Math.Min(31, modelName.Length));
```

#### B. IV Derivation (16 Bytes / 128 Bits)
The initial IV is derived from the `Version` attribute (e.g., `"1.0.5"` or `"1.0.5.2"`):
1. The version string is parsed into four integer components: `Major`, `Minor`, `Build`, and `Revision`.
2. Undefined components in the version string (such as `Revision` in `"1.0.5"`) default to `-1`.
3. Each component is converted to a 32-bit signed integer in little-endian format (4 bytes each).
4. The four components are concatenated sequentially:
   - Bytes 0-3: `Major`
   - Bytes 4-7: `Minor`
   - Bytes 8-11: `Build`
   - Bytes 12-15: `Revision`

For `"1.0.5"`, the parsed components are `Major=1`, `Minor=0`, `Build=5`, `Revision=-1`, which results in the following 16-byte array:
`\x01\x00\x00\x00 \x00\x00\x00\x00 \x05\x00\x00\x00 \xff\xff\xff\xff`

**C# Reference:**
```csharp
byte[] iv = new byte[16];
Array.Copy(BitConverter.GetBytes(version.Major), 0, iv, 0, 4);
Array.Copy(BitConverter.GetBytes(version.Minor), 0, iv, 4, 4);
Array.Copy(BitConverter.GetBytes(version.Build), 0, iv, 8, 4);
Array.Copy(BitConverter.GetBytes(version.Revision), 0, iv, 12, 4);
```

---

### 1.3 XML Encryption Standards
The XML files follow the standard **W3C XML Encryption Syntax and Processing** specification:

1. **Outer Encryption (`content: false`)**:
   The entire child element is serialized to a UTF-8 string and encrypted. The resulting base64 string is stored as a replacement element.
2. **IV Prepending**:
   When encrypting in CBC mode, standard XML Encryption prepends the 16-byte initialization vector to the beginning of the ciphertext. 
   - **On Encryption**: The payload stored in `<CipherValue>` is `Base64Encode(IV + Ciphertext)`.
   - **On Decryption**: The first 16 bytes of the decoded binary payload are extracted as the actual decryption IV, and the remaining bytes are decrypted using the derived `Key`.
3. **Encrypted Node Structure**:
   ```xml
   <EncryptedData Type="http://www.w3.org/2001/04/xmlenc#Element" xmlns="http://www.w3.org/2001/04/xmlenc#">
     <EncryptionMethod Algorithm="http://www.w3.org/2001/04/xmlenc#aes256-cbc" />
     <CipherData>
       <CipherValue>{base64_payload}</CipherValue>
     </CipherData>
   </EncryptedData>
   ```

---

### 1.4 Mapping to Linux Kernel `asus-armoury.h`
For Linux kernel development (specifically the `asus-armoury` driver), the decrypted ThrottleGear XML files contain the exact minimum (`LowerLimit`) and maximum (`UpperLimit`) bounds needed to populate `struct power_limits` entries.

#### A. CPU Power Limit Mapping
Inside `<ThrottlePluginCPUSettings>` -> `<OverclockItems>`:
- **`ppt_pl1_spl`** $\rightarrow$ maps to the `LowerLimit` and `UpperLimit` of **`<PPTLimit>`**.
- **`ppt_pl2_sppt`** $\rightarrow$ maps to the `LowerLimit` and `UpperLimit` of **`<fPPTLimit>`** or **`<APUsPPTLimit>`** depending on the CPU architecture (Intel PL2 / AMD sPPT).
- **`ppt_pl3_fppt`** $\rightarrow$ maps to the `LowerLimit` and `UpperLimit` of **`<fPPTLimit>`**.
- **`ppt_apu_sppt`** $\rightarrow$ maps to the `LowerLimit` and `UpperLimit` of **`<APUsPPTLimit>`**.
- **`ppt_platform_sppt`** $\rightarrow$ maps to the `LowerLimit` and `UpperLimit` of **`<PlatformsPPT>`**.

#### B. NVIDIA GPU Tunable Mapping
Inside `<ThrottlePluginGPUSettings>` -> `<NonSLIOverclockItems>`:
- **`nv_dynamic_boost`** $\rightarrow$ maps to the `LowerLimit` and `UpperLimit` of **`<DynamicBoost>`**.
- **`nv_temp_target`** $\rightarrow$ maps to the `LowerLimit` and `UpperLimit` of **`<NBThermalTarget>`**.
- **`nv_tgp`** $\rightarrow$ maps to the `Default` and limits defined under **`<TGPItem>`**.

---

## Part 2: Rust Tool Implementation

This section explains how the command-line utility implements cryptography and parses XML files.

### 2.1 Low-level Cryptography FFI
To maintain a lightweight footprint without heavy dependencies, the tool links directly against the system's pre-installed OpenSSL dynamic library (`libcrypto`) using Rust's Foreign Function Interface (FFI).

Using Rust's native `#[link(name = "crypto")]` and `extern "C"` declarations, the binary invokes low-level OpenSSL EVP functions:
- `EVP_CIPHER_CTX_new()` / `EVP_CIPHER_CTX_free()`: Allocate and free the cipher context structure.
- `EVP_aes_256_cbc()`: Fetch the cipher type structure for AES-256-CBC.
- `EVP_DecryptInit_ex()` / `EVP_EncryptInit_ex()`: Bind the context, cipher type, key, and IV.
- `EVP_DecryptUpdate()` / `EVP_EncryptUpdate()`: Process block updates.
- `EVP_DecryptFinal_ex()` / `EVP_EncryptFinal_ex()`: Flush the final block and perform PKCS7 padding verification/generation.

---

### 2.2 Custom XML Parsing & Serialization
Instead of utilizing large external crates, a custom recursive descent XML parser and serializer are implemented in Rust:
- An `XmlElement` structure represents XML elements, tracking their name, attributes, and child nodes (as a vector of `XmlNode` variants of elements or text).
- The serializer recursively prints the nodes using standard indentations and preserves namespaces exactly as specified in the original file format, preventing incorrect namespace prefixes.

---

## Part 3: License

This Rust implementation is distributed under the terms of the GNU Affero General Public License version 3 (AGPLv3). See the accompanying [LICENSE](../LICENSE) file for the full text.
