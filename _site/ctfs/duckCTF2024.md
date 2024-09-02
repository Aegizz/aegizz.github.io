# Duck CTF 2024

<img src="../ctfs/DSC02360.jpg">

The Duck CTF is a CTF hosted by both the University of Adelaide's Capture the Flag club and Computer Science club.

Whilst I didn't place as well as I had hoped (6th/20), it was still a ton of fun.

Big thanks to [Cuinn](https://www.linkedin.com/in/cuinnkemp/) and [Elke](https://www.linkedin.com/in/elke-milne) for being my teammates.

Here is a short write-up of a couple of the web challenges I did.

# Flag 1: Ducks&Tables
**Question**:

<img src="../ctfs/Pasted image 20240831204134.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

***Hint***: "There's nothing insecure about a simple database look-up tool - or is there?"

**Webpage**:

<img src="../ctfs/Pasted image 20240831204325.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

**Initial thoughts:**

So from the hint, we can probably guess this is some form of injection to get access to some flag stored in the database. Sweet, let's open up Burp.

**Burp:**

So firstly, I opened the Chromium in-built browser and navigated to the webpage. 

<img src="../ctfs/Pasted image 20240831204741.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

Then, I turned the intercept on and hit "Get Duck".

<img src="../ctfs/Pasted image 20240831205136.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

Ah, it seems that this text box is simply a way of sending a post request to the Webserver which queries the database for the kind of duck based on our input.

Let's send this to the repeater and have a little play around with the input.

<img src="../ctfs/Pasted image 20240831205708.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">
<img src="../ctfs/Pasted image 20240831205719.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

This input was a little optimistic, but Aha! It appears that we need to do some SQL Injection. The ending character of this string is `\"` so let's add this to our request to make it easier to inject. Now it should be pretty simple, let's boot up a favorite of mine, sqlmap. Let's change our JSON request to be `{"duck":"*"}` to let sqlmap know what to test and save our output to a text file `request3.txt`. Sqlmap does not support POST requests unless you specify and this is the easiest way to make sure the request is going where you want it to.

**SQLMAP:**

<img src="../ctfs/Pasted image 20240831210155.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

Sqlmap is a pretty handy tool; it'll try loads of different exploits and inputs to try and get the output you're looking for. I'm just going to give it my request, tell it to let it rip, and dump the entire database. If I was a better man, I'd look for a column where a flag might be hidden and ask sqlmap to dump columns where that input is not NULL, but I am not, so we're just going to ask it to get everything.
```bash
sqlmap -r request3.txt --level 5 --risk 3 --dump
```
After running this command, sqlmap quickly picked up on the database being SQLite, meaning we can restart the attack with the new database version set.

```bash
sqlmap -r request3.txt --level 5 --risk 3 --dump --dbms="sqlite"
```
After a couple of minutes, we can see that the ducks table has a secret TEXT. This is probably where the flag is stored; after a few minutes of being bored, I caved and used a where statement. <img src="../ctfs/Pasted image 20240901225246.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

```bash
sqlmap -r request4.txt --level 5 --risk 3 --dump --dbms="sqlite" --time-sec=2 --where "secret IS NOT NULL" --threads 4
```

After a few seconds, we get the following flag: <img src="../ctfs/Pasted image 20240901230005.png" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

Pretty easy, provided you know your way around web flags.

quack{s0_a_bl1nd_duck_w4lks_1nt0_a_t4bl3_4nd_s4y5_wh0_m0v3d_th3_p0nd?}

# Flag 2: Skullboard
**Question**:
<img src="../ctfs/Pasted image 20240901230318.png" alt="Question Image" style="display: block; margin: auto;" width="256"/>
***Do you have what it takes to out𝓯𝓻𝓮𝓪𝓴 the 𝓯𝓻𝓮𝓪𝓴𝓲𝓮𝓼𝓽 of all 𝓯𝓻𝓮𝓪𝓴𝓼?***

Yes, Yes I do.

Let's have a little investigation of the webpage and its provided code. 
### app.py
```python

# This is super secure, shush :x
def prepare_key(key):
    return jwt.utils.force_bytes(key)

jwt.api_jws._jws_global_obj._algorithms['HS256'].prepare_key = prepare_key

load_dotenv()

priv_key = open('priv.pem', 'r').read()
pub_key = open('pub.pem', 'r').read()


def users_only(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.cookies.get('token')
        if not token:
            return redirect(url_for('login'))
        
        try:
            # This is kinda janky but it works
            alg = jwt.get_unverified_header(token)['alg']
            jwt.decode(token, pub_key, algorithms=alg)
        except:
            return redirect(url_for('login'))
        
        return f(*args, **kwargs)

    return decorated


@app.route('/skullboard', methods=['GET'])
@users_only
def skullboard():
    token = request.cookies.get('token')

    try:
        # it works so why fix it :^)
        alg = jwt.get_unverified_header(token)['alg']
        data = jwt.decode(token, pub_key, algorithms=alg)
        if data['user'] == 'admin':
            return render_template('skullboard.html', message=f'Hœh!? How did you get in here?? {os.getenv("FLAG")}')
    except jwt.ExpiredSignatureError:
        return redirect(url_for('login'))
    except jwt.InvalidTokenError:
        return redirect(url_for('login'))
    
    return render_template('skullboard.html', message='You must be the admin user in order to view the secret!')


if __name__ == '__main__':
    app.run(host="0.0.0.0", port="8000", debug=False)
```
So this server will provide us with a flag when our JSON Web Token (JWT) defines our user as an admin. Hmmmm, this is a little tricky, there are some hints in the code let's go from there. It seems that when we send a GET request to the /skullboard page, we can define our algorithm in the JWT. This... does not really help much. We need to find some way of self-signing these JSON Web Tokens. There is a hint right at the top of the code—it seems that whoever developed this application overwrote the JWT HS256 algorithm with their own custom algorithm. Let's research a bit more on some tricks we can do with JWTs.

Whenever I feel the need to do some research on a given methodology or research on an area I am unfamiliar with, my go-to is Hacktricks.

<img src="../ctfs/Pasted%20image%2020240901232503.png" alt="Hacktricks JWT Research" />

Hacktricks gave me a few ideas to work with. Firstly, they recommend jwt_tool, a Python script for testing for JWT vulnerabilities and configurations.

Let's get started. First, let's make an account and get a token.

<img src="../ctfs/Pasted%20image%2020240901232742.png" alt="Account and Token Creation" style="display: block; margin-left: auto; margin-right: auto;" width="256" />

After logging in with this account, I checked my cookies and found this token:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZnJlYWtpZXN0RnJlYWsiLCJleHAiOjE3MjUyMDI2Njl9.avknANuiNBIkPzQ4m-4RXy9XEbB2CHVxK4ZRoBo__Y8_2QtI9wpG6iLUqnvMnm1Mp0rSZ2Y83tsQx-7ckVw3BLVqhFVMP5Yd39GIoxcBfyNYa8L0dlAd4M9SK4nf9Yfyu3O_LHjni1Lz1G_PcxLTODB87hA69nro-Dgb6YT-frD97B6nbANyB0rtEgfOceeAZ8YM7PkTIasCyCHDRpkqgvzPpHHTepZklw8WF0JUYW62w0PGMf8h24NBZjKRrhW7TTQ2WP0Vmo7XxLG9uNnTrD3qNKX_edzGrQ8HzI3b95Shyxu36XgKtTbm46PxlYshtPJmumprVwz3nqbq-Anu9w
```

Let's run this with the tool recommended by Hacktricks.

<img src="../ctfs/Pasted%20image%2020240901233021.png" alt="Running JWT Tool" style="display: block; margin-left: auto; margin-right: auto;" width="256" />

This tool shows that the default algorithm appears to be RS256 and that the user value is stored in the base64 encoded values of this JWT. I ran a few of the test commands on this tool and didn't seem to get any quick wins back. Alright, let's do a little more research.

Hacktricks recommends an exploit where, by changing the algorithm from RS256 to HS256, a program reuses the public key as the private key used in signing an HS256 token.

<img src="../ctfs/Pasted%20image%2020240901234914.png" alt="Algorithm Change Exploit" />

This... just might work. I found this nice tool online that can use the public key from my JWT here to create a new payload.

<img src="../ctfs/Pasted%20image%2020240901235837.png" alt="JWT Public Key Tool" />

Alright, if we can find a way to get the public key somehow, we should be able to exploit this coding mistake. This GitHub repo should do the trick: https://github.com/silentsignal/rsa_sign2n. Here we run the x_CVE-2017-11424.py script. This provides us with the public key of the JWTs.

<img src="../ctfs/Pasted%20image%2020240902025125.png" alt="Running RSA Sign Script" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">
<img src="../ctfs/Pasted%20image%2020240902025446.png" alt="RSA Public Key Extracted" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

Here, from our encoded token, we can copy this public key, encode it in base64, and use it to generate an HS256 JWT.

<img src="../ctfs/Pasted%20image%2020240902025510.png" alt="Base64 Encode Public Key" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">
<img src="../ctfs/Pasted%20image%2020240902025706.png" alt="Generated HS256 JWT" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

Sweet! Let's try and reload the Skullboard with this encoded token:

`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiYWRtaW4iLCJleHAiOjE4MjUyMDI2Njl9.b8r4rnwIi1Km4QPpVJZy6z3Z-wMdLz8q13UrejFe54s`

<img src="../ctfs/Pasted%20image%2020240902025902.png" alt="Final Flag" style=" display: block;
  margin-left: auto;
  margin-right: auto;
  width: 50%;">

Oh yeah! Are we freaky or what! Here's our flag: `quack{uns4n1t1s3d_4lg0_m4k3s_au7h_byp4ss_3zpz!1!1}`. This was a really cool box, big thanks to Samiko for creating these challenges and helping me ensure the keys I generated were correct.

If you need help getting this working on your machine, feel free to contact me on Discord: `aegizz.` Happy hacking!