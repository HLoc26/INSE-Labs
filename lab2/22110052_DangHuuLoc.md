# Task 1: Public-key based authentication 
**Question 1**: 
Implement public-key based authentication step-by-step with openssl according the following scheme.
![alt text](image-1.png)

**Answer 1**:

## Step 1: Preparing the environment
### 1.1. Create docker VMs
I have created 2 docker container named lab2-server for the server machine and lab2-client for the client machine:

![image](https://github.com/user-attachments/assets/9f87fce3-d9a6-415b-9867-7c038bdab777)

### 1.2. Install nessesary packages
Then, on both machine, install the nessesary packages:

1. `openssl`: To use openssl
2. `netcat` (traditional): To send and receive files, specifically the public key.
3. `net-tools`: To use the command `ifconfig` to see the IP address of the machine.
4. `xxd`: To inspect hex values from binary files

Command: `apt update && apt install openssl netcat-traditional net-tools xxd`

## Step 2: Create client's private and public keys
### 2.1. Create private key

I used RSA to encrypt the private key, with the length of 512 bits for faster computational speed (since this is just a lab, the length should be 2048 bits or more in real life)

Command: `openssl genrsa -out client_private.pem 512`

Explain the command:

- `genrsa`: Generate RSA private key.
- `client_private.pem`: The file name to save the private key in.
- `512`: The length of the private key.

The private key is saved in file named `client_private.pem`

![image](https://github.com/user-attachments/assets/80b499fa-061e-44bf-86b2-eea52b2ac933)

### 2.2. Create public key

From the private key, we can generate the public key using the following command:

```
openssl rsa -in client_private.pem -pubout -out client_public.pem
```

The above command uses the `client_private.pem` as input, and extract a public key from the private key (`-pubout`) and save the public key to `client_public.pem` (`-out client_public.pem`)

![image](https://github.com/user-attachments/assets/4ad5c396-cf85-4dd4-9562-2846fec0beac)

### 2.3. Send public key to server machine

In order to send the public key `client_public.pem` to the server machine, we use `netcat`.

First, on the server, we have to know the IP address of the machine:

![image](https://github.com/user-attachments/assets/584567f5-0974-41fd-b41e-75f776745a0a)

The IP address of the server is `172.17.0.2`

First, the server have to be listening on a port (I used port 4444) using the command 

```
nc -lv -p 4444 > client_public.pem
```

![image](https://github.com/user-attachments/assets/27c189de-fc06-4dc1-985f-60094e773707)

The server machine will be listening (`-l`) on port 4444 (`-p 4444`), and will print out the log (`-v`). The receive bits will be saved into `client_public.pem` on the server machine.

Then, the client can send the file using
```
nc 172.17.0.2 4444 < client_public.pem
```
The client machine will send the file `client_public.pem` to the server's IP address, port 4444

![image](https://github.com/user-attachments/assets/8f87de58-3585-425b-ae42-702cfa5fbfc2)

Now, the client_public is now on the server machine

![image](https://github.com/user-attachments/assets/6ea07a03-637a-4918-b1a3-ab5cbc2e39e8)

We can see that the public key is the same on both machine. The public key is sent successfully.

## Step 3: Encrypt the message

First, create a `challenge.txt` file
```
echo "Hello from the server" > challenge.txt
```

And encrypt the message using the following command
```
openssl pkeyutl -encrypt -inkey client_public.pem -pubin -in challenge.txt -out challenge.enc
```

The command will use RSA to encrypt the file `challenge.txt` using the public key from `client_public.pem`. The result will be saved in `challenge.enc`.

![image](https://github.com/user-attachments/assets/6db1281e-bcbf-47eb-b562-e1eda5e3b0c8)

## Step 4: Send the encrypted message to the client.

Note that on [step 2.3](#23-send-public-key-to-server-machine), the server received the `client_public.pem` from the IP address `172.17.0.3`, which is the IP address of the client machine. The server will send the encrypted message to that address.

First, the client must be listening on port 4445, then the server can send the file.

The commands are similar to [step 2.3](#23-send-public-key-to-server-machine).

![image](https://github.com/user-attachments/assets/66fab021-26cb-4025-af27-0ae5d696c1f2)

The client now have the encrypted challenge file:

![image](https://github.com/user-attachments/assets/d0a966cf-8961-4afe-84e0-6d48d1ea1d4d)


## Step 4: Decrypt the challenge

Using the decrypt command:
```
openssl pkeyutl -decrypt -inkey client_private.pem -in challenge.enc -out decrypted_challenge.txt
```

The command is used to decrypt the encrypted `challenge.enc` using the client's private key (`-inkey client_private.pem`), and save the decrypted to `decrypted_challenge.txt`.
 
![image](https://github.com/user-attachments/assets/3a3c7d1a-38ab-44bd-a9d9-20fe70e9c6a9)

## Step 5: Sign the challenge using client's private key

I will use -sha256 to create the signature to sign using client's private key. The result is saved to `signed_challenge.bin`.
```
openssl dgst -sha256 -sign client_private.pem -out signed_challenge.bin decrypted_challenge.txt
```

![image](https://github.com/user-attachments/assets/27b980e0-ce97-40e5-a4a9-e50c694c8611)

## Step 6: Send the signature back to the server

Similar to [step 2.3](#23-send-public-key-to-server-machine), I will now send the signature to the server.

![image](https://github.com/user-attachments/assets/1c7710ec-4e61-44bd-a81e-dc5492163548)

![image](https://github.com/user-attachments/assets/46d5e22f-5cc1-4db6-9ed0-46017e12514a)

## Step 7: Verify the signature

```
openssl dgst -sha256 -verify client_public.pem -signature signed_challenge.bin challenge.txt
```

Explain the command:

- `-sha256`: Use SHA-256 for hashing.
- `-verify client_public.pem`: Verify the signature using the public key in client_public.pem.
- `-signature signed_challenge.bin`: Specifies the received `signed_challenge.bin` as the signature file to check.
- `challenge.txt`: The file to verify.

![image](https://github.com/user-attachments/assets/f6573eb5-25bb-4cab-b94c-f04d4b8a314f)

The file is now verifed.

**Conclusion:** I have successfully implemented public-key based authentication step-by-step with `openssl`.

# Task 2: Encrypting large message
Create a text file at least 56 bytes.

### Prepare the text file
I used Python (in my own machine) to print 56 characters, then saved it into `large.txt` in the VM.
![image](https://github.com/user-attachments/assets/1edf8f0d-7c05-4105-bb12-9a84426e2f54)

### Generate 

## Question 1:
Encrypt the file with aes-256 cipher in CFB and OFB modes. How do you evaluate both cipher as far as error propagation and adjacent plaintext blocks are concerned. 
### Answer 1:

The command to encrypt the file:
```
openssl enc <algorithm> -in <input-file> -out <output-file> -k <passkey> -iv <iv>
```

**Encrypt the file with AES-256 in CFB mode**
```
openssl enc -aes-256-cfb -in large.txt -out message_ofb.enc -k secretpassword -iv 123
```
Result:

![image](https://github.com/user-attachments/assets/54e838cf-b651-4400-9407-2ee0ef13ca4b)

- Error propagation: In CFB mode, if a single byte of the ciphertext is corrupted, it affects the decryption of that corrupted byte and the following byte only. The error propagates only to the extent of the next byte.
- Adjacent plaintext blocks: CFB mode operates on segments of the plaintext, where the size of segments is often the block size of the cipher (like 128 bits for AES). Corruption in one segment does not affect all subsequent segments.

**Encrypt the file with AES-256 in OFB mode**:
```
openssl enc -aes-256-ofb -in large.txt -out message_ofb.enc -k secretpassword -iv 123
```

Result:

![image](https://github.com/user-attachments/assets/b9f8df66-2b0e-49ae-a415-2a819db450f0)

- Error propagation: In OFB mode, a single-byte error in the ciphertext only affects the decryption of that particular byte. The error does not propagate, making OFB more resilient to errors compared to CFB.
- Adjacent plaintext blocks: OFB mode is a stream cipher mode, so corruption in the ciphertext does not impact adjacent plaintext blocks. Each byte is independently encrypted using the keystream.

## Question 2:
Modify the 8th byte of encrypted file in both modes (this emulates corrupted ciphertext).
Decrypt corrupted file, watch the result and give your comment on Chaining dependencies and Error propagation criteria.

### Answer 2:
For this question, I will install `hexedit` to modify the encrypted file: `apt install hexedit`

## Modify the `large_cfb.enc` file
Command: `hexedit message_ofb.enc`

Before:

![Untitled](https://github.com/user-attachments/assets/21adfac3-3868-42fe-a187-4c756c4e6ecc)

After (from `0x5F` to `0x53`):

![Untitled](https://github.com/user-attachments/assets/b830442f-5200-4723-8ec4-0e13af0f3994)


## Modify the `large_ofc.enc` file

Command: `hexedit message_ofb.enc`

![Untitled](https://github.com/user-attachments/assets/cf0dcbf7-33a8-4d2a-8b78-c16c9e15676a)

![Untitled](https://github.com/user-attachments/assets/fc71d0b5-7f10-4371-ae3b-a6c9f5abe9f6)


## Decrypt corrupted files

Command:
```
openssl enc -d <algorithm> -in <corrupted-file> -out <output-file> -k <secret-key> -iv <iv>
```

**Decrypting large_cfb.enc**:
```
openssl enc -d -aes-256-cfb -in large_cfb.enc -out cfb.txt -k secretpassword -iv 123
```

