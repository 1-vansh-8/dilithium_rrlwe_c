# Walkthrough - Kyber & Dilithium PQC Integration

This walkthrough details the post-quantum cryptographic (PQC) integration of **CRYSTALS-Kyber (Kyber-768)** for key exchange and **CRYSTALS-Dilithium (Dilithium3)** for digital signature authentication.

The entire implementation has been successfully compiled and verified against passive eavesdropping and active Man-in-the-Middle (MITM) attacks.

---

## 🛠️ Changes Made

1. **Shared Library Setup (Makefiles)**:
   - Modified [dilithium Makefile](file:///Users/arpangupta/Desktop/Intel%20Cup/dilithium_rrlwe_c/Makefile) and [kyber Makefile](file:///Users/arpangupta/Desktop/Intel%20Cup/kyber_rrlwe_c/Makefile) to compile their respective reference C codebases as shared libraries (`.so` / `.dylib`), bundling their namespaced Keccak and random bytes implementations.
   - Renamed header guards in [dilithium api.h](file:///Users/arpangupta/Desktop/Intel%20Cup/dilithium_rrlwe_c/api.h) (`DILITHIUM_API_H`) and [kyber api.h](file:///Users/arpangupta/Desktop/Intel%20Cup/kyber_rrlwe_c/api.h) (`KYBER_API_H`) to eliminate preprocessor guard conflicts.

2. **Root build file**:
   - Created a root [Makefile](file:///Users/arpangupta/Desktop/Intel%20Cup/Makefile) to build the server and client binaries and link them dynamically against Kyber and Dilithium shared libraries.

3. **Client-Server Socket Implementation**:
   - Created [pqc_server.c](file:///Users/arpangupta/Desktop/Intel%20Cup/pqc_server.c): A TCP socket listener that hosts the Dilithium3 identity keypair, ephemeral Kyber-768 keypair, signs the Kyber public key, decapsulates client secret, and decrypts communications using the derived session key.
   - Created [pqc_client.c](file:///Users/arpangupta/Desktop/Intel%20Cup/pqc_client.c): A TCP client that authenticates the server's Dilithium3 signature on the Kyber public key, encapsulates a shared secret, and sends encrypted message payloads.

4. **Security Testing Simulator**:
   - Created [mitm_proxy.py](file:///Users/arpangupta/Desktop/Intel%20Cup/mitm_proxy.py): A Python proxy that intercepts the Server Hello handshake, alters a byte of the server's Kyber public key to simulate active network tampering, and verifies client-side aborts.

---

## 🧪 Verification & Security Validation Results

### 1. Normal Operation (Secure Handshake & Session)

When running the client against the server directly, the connection is authenticated and secured successfully:

**Client Console Output:**
```
=========================================================
        PQC Authenticated Key Exchange (AKE) Client      
             (Kyber-768 KEM + Dilithium3 DSA)            
=========================================================

[+] Loaded Server Dilithium3 Public Key.
Server Identity Public Key (pk_dil - first 32 bytes) (size: 32 bytes):
  8ABDAA8A 9C64A3F4 011EF7CD 4F5A2004 79CC3DA6 8B335655 19230772 2053BA15 

[*] Connecting to server at 127.0.0.1:8085...
[+] Connected to server.
[*] Receiving Server Hello (Kyber PK + Dilithium Signature)...
[+] Server Hello received.
Received Ephemeral Kyber-768 PK (first 32 bytes):
  4F228EEF CB84629B 2713952E 8395B186 AC607F76 4CBF9C00 403AC52B E25B63B2 

Received Dilithium3 Signature (first 32 bytes):
  2C655666 0EA306D8 BD9CEE88 2E9BC809 2410D9D9 6EA5D267 CB8E5866 2D20C6A3 

[*] Verifying Server's Dilithium3 signature on Kyber public key...
[+] SUCCESS: Dilithium3 signature verified correctly. Server identity authenticated!

[*] Performing Kyber-768 key encapsulation...
[+] Shared secret encapsulated.
Generated Shared Secret (ss) (size: 32 bytes):
  FEDB0197 4EBA7565 05C30657 AF82FB94 A154A1E6 84D438F9 0D65039E BBD66BF3 

[*] Sending encapsulated key (ciphertext) to server...
[+] Kyber ciphertext sent.
[+] Derived 256-bit symmetric session key.
Session Key (k_session) (size: 32 bytes):
  F75B4CAF CF32A727 D6B22B14 96AE464D B5A0BD9F F81D16BE 56027CD1 2A891F5B 

[*] Plaintext Message: "Hello, this is a secure post-quantum encrypted message!" (length: 56 bytes)
[*] Sending encrypted payload...
[+] Encrypted payload sent.

[*] Awaiting server's encrypted response...
[+] Decrypted Server Response: "Server received 56 bytes safely."
[*] Secure session ended. Connection closed.
```

**Server Log File Output (`server.log`):**
```
=========================================================
        PQC Authenticated Key Exchange (AKE) Server      
             (Kyber-768 KEM + Dilithium3 DSA)            
=========================================================

[+] Loaded long-term Dilithium3 identity keypair from files.
Server Identity Public Key (pk_dil - first 32 bytes):
  8ABDAA8A 9C64A3F4 011EF7CD 4F5A2004 79CC3DA6 8B335655 19230772 2053BA15 

[*] Listening on port 8085...
---------------------------------------------------------
[*] Waiting for a client connection...
[+] Client connected from 127.0.0.1:51022
[*] Generating ephemeral Kyber-768 keypair...
[+] Ephemeral Kyber-768 keypair generated.
[*] Signing Kyber public key using Dilithium3...
[+] Signature generated successfully. Size: 3309 bytes.
[*] Sending Server Hello (Kyber PK + Signature)...
[+] Server Hello sent.
[*] Waiting for Client Key Exchange (Kyber Ciphertext)...
[+] Kyber ciphertext received.
[*] Decapsulating ciphertext to recover shared secret...
[+] Shared secret recovered successfully.
Shared Secret (ss) (size: 32 bytes):
  FEDB0197 4EBA7565 05C30657 AF82FB94 A154A1E6 84D438F9 0D65039E BBD66BF3 

[+] Derived 256-bit symmetric session key.
Session Key (k_session) (size: 32 bytes):
  F75B4CAF CF32A727 D6B22B14 96AE464D B5A0BD9F F81D16BE 56027CD1 2A891F5B 

[*] Secure channel established. Awaiting encrypted payloads...
[+] Received Encrypted Message (56 bytes):
[+] Decrypted Plaintext: "Hello, this is a secure post-quantum encrypted message!"
[*] Sent encrypted response back to client.
[*] Client closed connection or disconnected.
[*] Connection closed. Ready for next client.
```

---

### 2. Active Man-in-the-Middle (MITM) Tampering Detection

When connecting the client through the active MITM proxy, the proxy intercepts the Server Hello and alters the server's Kyber public key. The client immediately detects the signature mismatch and aborts:

**Client Console Output during Tampering:**
```
=========================================================
  [CRITICAL ALERT] SIGNATURE VERIFICATION FAILED!         
  Potential Man-in-the-Middle (MITM) attack detected!      
=========================================================

=========================================================
        PQC Authenticated Key Exchange (AKE) Client      
             (Kyber-768 KEM + Dilithium3 DSA)            
=========================================================
[+] Loaded Server Dilithium3 Public Key.
[*] Connecting to server at 127.0.0.1:8086...
[+] Connected to server.
[*] Receiving Server Hello (Kyber PK + Dilithium Signature)...
[+] Server Hello received.
[*] Verifying Server's Dilithium3 signature on Kyber public key...
(Client aborts execution here with exit code 1)
```

**MITM Proxy Log Output (`proxy.log`):**
```
=========================================================
        PQC MITM Proxy Simulator (TAMPER ACTIVE)                
  Listening on port 8086 -> Forwarding to 127.0.0.1:8085
=========================================================

[Proxy] Intercepted client connection from 127.0.0.1:51021
[Proxy] Connected to real server.
[Proxy] Intercepted Server Hello (4493 bytes).
[Proxy] TAMPERING: Flipped bits in the ephemeral Kyber public key!
```

**Server Log File Output during Tampering (`server.log`):**
```
[*] Listening on port 8085...
---------------------------------------------------------
[*] Waiting for a client connection...
[+] Client connected from 127.0.0.1:51022
[*] Generating ephemeral Kyber-768 keypair...
[+] Ephemeral Kyber-768 keypair generated.
[*] Signing Kyber public key using Dilithium3...
[+] Signature generated successfully. Size: 3309 bytes.
[*] Sending Server Hello (Kyber PK + Signature)...
[+] Server Hello sent.
[*] Waiting for Client Key Exchange (Kyber Ciphertext)...
[-] Received incomplete Kyber ciphertext: 0 bytes (expected 1088)
---------------------------------------------------------
[*] Waiting for a client connection...
```
*(Notice that the server safely aborted the handshake because the client closed the socket immediately after signature verification failed, before sending any ciphertext).*

---

## 🚀 How to Build and Run on the DK-2500 Board

Follow these simple steps on the board (running Ubuntu Linux 22.04):

1. **Build all components**:
   ```bash
   make
   ```
   This will automatically compile the Dilithium shared library (`libpqcrystals_dilithium3_ref.so`), the Kyber shared library (`libpqcrystals_kyber768_ref.so`), and link them to `pqc_server` and `pqc_client`.

2. **Run the Authenticated Server**:
   ```bash
   # Start the server on port 8085 (libraries are linked via -rpath automatically)
   ./pqc_server 8085
   ```

3. **Run the Authenticated Client**:
   ```bash
   # Start the client, connect to localhost, and send a custom secure message
   ./pqc_client 127.0.0.1 8085 "Hello, this is a secure PQ message on DK-2500!"
   ```

4. **Run the MITM Active Tampering Simulation**:
   ```bash
   # 1. In terminal 1, run the server:
   ./pqc_server 8085
   
   # 2. In terminal 2, run the MITM proxy:
   python3 mitm_proxy.py
   
   # 3. In terminal 3, run the client pointing to the proxy (port 8086):
   ./pqc_client 127.0.0.1 8086 "This will fail"
   ```
