+++
title = 'Lsass Extractor'
date = 2023-09-16T16:39:49+02:00
draft = false
image = '/posts/lsass-extractor/static/cover.png'
+++

## Introduction

In TeamFence, we strongly believe that nowadays it's increasingly important for an offensive operator to understand the
details behind the techniques and tools.

For this reason, in this blog post, we will delve into how Mimikatz extracts credentials from lsass dumps, enabling us
to then re-implement the logic into a tool tailored to our future needs.

**Note**: all the analyses conducted throughout this project have been made on a Windows 11 Enterprise Evaluation
workstation.

## Why LSASS?

Before diving into the technical details, it's important to ask: Why are we doing this?

When a hacker gains access to a target machine running Windows, one of the crucial initial steps involves obtaining
credentials. Windows offers multiple potential targets, but two primary repositories stand out: the local security
database (Security Account Manager) and the LSASS (Local Security Authority Subsystem Service) process.

In this post we are going to work on LSASS, which serves as a pivotal component within Windows, overseeing
authentication. As the central hub for
authentication requests from various services, it plays a pivotal role in streamlining the authentication flow. This
process implements various authentication
packages such as NTLM, Kerberos, WDigest, among others. Consequently, it presents an intricate, valuable, and
information-rich target
for hackers.

## NTLM Hashes

As we saw in the previous section, LSASS handles several authentication packages, including NTLM. NTLM is a
challenge-response authentication protocol that is widely used in Windows environments. When a user logs into a Windows
machine, the system stores the user's credentials in memory. This information includes the user's password in the form
of an NTLM hash.

In this post, we will focus on extracting these NTLM hashes from the LSASS process, which will allow us to understand
the underlying logic in a easier way.

In the following image we can see the internal LSASS structure, highlighting where the NTLM hashes are stored.

![lsass](/posts/lsass-extractor/static/lsass-structure.png)

## Where are all the pieces?

To start, it's crucial to understand where the information is stored and in what format. The LSASS process is a critical
component, and not all the information is stored in plain text. In this case, the NTLM hashes are encrypted. So, to
extract them, we need to locate the keys for decryption. The following diagram illustrates which modules are involved in
the handling of the keys and the credentials.

<img src="/posts/lsass-extractor/static/information-schema.png" alt="where are the pieces" style="display: block; margin-left: auto; margin-right: auto; width: 500px; height: auto;">

It's important to remember that the cryptographic material of the LSASS process is runtime data, meaning it changes
after every reboot. Consequently, we must extract them each time to ensure up-to-date information.

```text
"It is interesting to note that the encryption management is carried out through LsaProtectMemory, which is a wrapper
for LsaEncryptMemory, which in turn is a wrapper for BCryptEncrypt, which is a common encryption function present in
bcrypt.h"
```

## Modules templates

At this point we have a clear understanding in which area of the memory all the pieces are stored. Now we need to find a
way to extract them.
By analyzing the code of PyPyKatz, a Python version of Mimikatz, we can understand the approach used to extract the keys
and the credentials.

First of all we need to extract some information about the system: ProcessorArchitecture, BuildNumber, and from the "
ModuleListStream" the TimeDateStamp of "lsasrv.dll".

Once we have this information, we can use it to identify the correct template to use for the extraction. The templates
are classes that contain the signature and offsets that allow us to know how to move within the memory to extract the
keys and the credentials.

In our case, which is a Windows 11 Enterprise Evaluation, the template for the lsasrv.dll part is the following:

```python
class LSA_x64_6(LsaTemplate_NT6):
	def __init__(self):
		LsaTemplate_NT6.__init__(self)
		self.key_pattern = LSADecyptorKeyPattern()
		self.key_pattern.signature = b'\x83\x64\x24\x30\x00\x48\x8d\x45\xe0\x44\x8b\x4d\xd8\x48\x8d\x15'
		self.key_pattern.IV_length = 16
		self.key_pattern.offset_to_IV_ptr = 67
		self.key_pattern.offset_to_DES_key_ptr = -89
		self.key_pattern.offset_to_AES_key_ptr = 16
				
		self.key_struct = KIWI_BCRYPT_KEY81
		self.key_handle_struct = KIWI_BCRYPT_HANDLE_KEY
```

Take a look at
the [source code](https://github.com/skelsec/pypykatz/blob/c91dcdc09289ad2e93c475e7c640d0f90906a7c0/pypykatz/lsadecryptor/lsa_template_nt6.py#L301)
of PyPyKatz to see all the templates.

## Initialization vector extraction

In order to extract the keys, we need to extract the Initialization Vector (IV) first. The IV is a random number used to
ensure that the same plaintext message does not result in the same ciphertext. The IV is stored in the memory, and we
can extract it by looking in the memory of the lsasrv.dll module for the pattern present in the template.

Once we have the location of the pattern, we can use the offsets to move within the memory and extract the IV.

<img src="/posts/lsass-extractor/static/iv-schema.png" alt="schema to extract the IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

As we can see from the schema, once we have the location of the pattern, we have to jump of a prefixed offset (present
in the template) of 67 bytes to find another value, which is another offset. This offset is the location of the IV.

## Keys extraction

Now is the time to extract the keys. The approach is similar to the one used to extract the IV. We need to find the same
pattern in the memory and then use the offsets to extract the keys.

<img src="/posts/lsass-extractor/static/des-schema.png" alt="schema to extract the IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

This time, using the pointer doesn’t directly lead to the desired value, but rather to a BCRYPT_KEY_HANDLE struct,
which, in bcrypt.h, is a data type that enables referencing and manipulating cryptographic keys within the CNG
framework. In pypykatz, the class KIWI_BCRYPT_HANDLE_KEY replicates this type.

```python
class KIWI_BCRYPT_HANDLE_KEY :
    def __init__ ( self , reader ) :
        self.size = ULONG( reader ).value
        self.tag = reader.read(4) #'UUUR'
        self.hAlgorithm = PVOID(reader).value
        self.ptr_key = PKIWI_BCRYPT_KEY(reader)
        self.unk0 = PVOID (reader).value
[...]
```

At this point, we have all the necessary information to decrypt the NTLM hashes. We have the IV and the keys. We can now
start the research of the LogonSessions.

## LogonSessions

The LogonSessions are the structures that contain the information about the users that are logged into the system. The
LogonSessions are stored in the memory of the Msv1_0.dll module, and we can extract them by looking for the pattern
present in the specific template, which in our case is the following:

```python
elif WindowsBuild.WIN_11_2022.value <= sysinfo.buildnumber < WindowsBuild.WIN_11_2023.value: #20348
    template.signature = b'\x45\x89\x34\x24\x4c\x8b\xff\x8b\xf3\x45\x85\xc0\x74'
    template.first_entry_offset = 24
    template.offset2 = -4
```

Is not possible to know the exact number of LogonSessions that are present in the memory, so we need to retrieve the
counter first.

#### LogonSessions counter and addresses

With a similar approach to the one used to extract the IV and the keys, we need to find the pattern in the memory and
then use the offsets to extract the counter.

This counter allow us to iterate over all the LogonSessions address, where every entry is 16 bytes next to the previous.

<img src="/posts/lsass-extractor/static/counter-addresses.png" alt="schema to extract the IV" style="display: block; margin-left: auto; margin-right: auto; width: 600px; height: auto;">

By casting those addresses to a char*, they become a pointer that points to the memory location where the LogonSessions
entry are stored. This grants the ability to access the data associated with the LogonSession, enabling retrieval of
relevant information.

#### Username and Domain

Now that we have the addresses of the LogonSessions, we can start to extract the information. Jumping to the offset of
144 bytes from the address, several fields are present, including the username length, the max length, and a pointer to
the actual username.
The same logic applies to the domain, which is stored in the memory after the pointer to the username.

Now it is possible to extract the username and the domain of the user by following the pointers and reading from the
memory the exact number of bytes that are stored in the max length field.

<img src="/posts/lsass-extractor/static/logon-structure.png" alt="schema to extract the IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

#### Credential List

For the final part of the extraction, we need to focus on the credential lists. Starting from the pointerToDomain saw in
the previous section, by advancing an offset of 96 bytes, the memory address
where the first
'Credential List' is located can be found. Once the value at the pointed address is read as an uint64_t and subsequently
interpreted as a memory address, it will provide the memory address where the second 'Credential List' is situated, and
this process continues iteratively. Since it is a circular linked list, when the value matches the memory address of the
first struct, it indicates that the entire list has been traversed.

Every entry in the linked list of 'Credential List' contains its own associated linked list known as the 'Primary
Credential List'. To retrieve the address of the first entry in this linked list, it is necessary to advance by an
offset of 16 bytes and read this memory area as a char pointer.

The following schema illustrates the structure of the lists:
<img src="/posts/lsass-extractor/static/credlist-struct.png" alt="schema to extract the IV" style="display: block; margin-left: auto; margin-right: auto; width: 650px; height: auto;">

Within this final structure, the encrypted NTLM hash can be found. Much like the linked list of 'Credential List',
interpreting the number at the address of the first entry of the Primary Credential as an address leads to the discovery
of the subsequent struct, following a circular list. By following the structure, it is possible to retrieve
the size of the encrypted NTLM and the memory address where this NTLM is located, allowing for its subsequent reading
and
extraction.

## NTLM decryption

The encryption algorithm used to encrypt the hash cannot be determined in advance. To identify the algorithm, the
encrypted NTLM is divided by 8, and its remainder is checked to see if the NTLM length is divisible by 8. If the NTLM
length is evenly divisible by 8, AES encryption is employed. Conversely, if the NTLM length is not divisible by 8,
indicating an irregular length, 3DES encryption is utilized.

The following pseudocode represents the algorithm selection logic:

```c
if ( encryptedNTLMLength % 8 != 0) {
 // Encrypted cred was empty ?
} else {
 // Prepare algorithm functions and IV
 if ( encryptedNTLMLength % 8) {
    // Perform AES decryption using the provided key
 } else {
    // Perform 3 DES decryption using the provided key
 }
}
```

Once the algorithm used has been identified, the remaining task is to apply the acquired information using the bcrypt.h
library, following the same approach as MSV. By configuring the required variables to utilize the correct algorithm and
subsequently invoking the BCrypt- Decrypt function with the encrypted key and the recovered IV, it becomes possible to
retrieve the plaintext NTLM hash.