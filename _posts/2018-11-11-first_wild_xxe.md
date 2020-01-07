---
layout: post
title: "My first XXE in the wild"
---

*Disclaimer: This information was found during a pen-test on a client. For that reason, my explanations and examples will be very vague and contain no sensitive information relative to the target but it will be kept close to the actual finding.*

If you’re not sure what XXE is, please refer to OWASP’s wiki page.

-----

## Initial finding

Whilst enumerating the target web app, various features were found that allowed authenticated users to upload their own data. An example of an XML template was obtainable via a download and presented the correct method of including custom data that the application could parse. As soon as I saw that XML was usable I started crafting small payloads just to test for XXE vulnerabilities.

I initially tried to read a file with a name that definitely wouldn’t exist to see if I could leak any information about the target environment. 

The redacted XML file I used was:
 
``` 
<?xml version="1.0" encoding="ISO-8859-1"?>
  <!DOCTYPE file [  
   <!ELEMENT file ANY >
    <!ENTITY xxe SYSTEM "file://_definitely_doesnt_exist_asdfasdf" >
  ]>
  <root>
    <name>&xxe;</name>
    [Redacted fields]
  </root> 
```

After uploading the crafted file, an error was printed within a field that would have been populated by the XML node field, proving my XXE payload had triggered and was returning data. The error was a ‘File Not Found’ error and included the whole file-path to the current location. Ultimately this information leak provided me with the following information:

* I was targeting a Linux environment
* I had an XXE vulnerability to leverage
* I effectively had RCE

With these 3 points I was able to start leaking sensitive files such as /etc/passwd, configuration files and source code that would only normally be utilised by the server to help potentially find other weaknesses and issues within the target.

What came next?
I crafted a short python script that performed the XXE vulnerable request over and over again. Each iteration attempted to obtain a different file from a dictionary I provided. Ultimately, I had created a brute force tool, similar to “dirbuster”, but specifically for enumerating the internal files and directories of my target. Using this quick tool lead to leaking the AWS secret keys... game over \o/

The use of this tool probably wasn't the most efficient approach, but it worked in this case. However, this accompanied by a 'Billion Laughs' attack lead to some issues for the client's environment. Great news for the report though ;)