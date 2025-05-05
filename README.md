Analyzing the Web Page
Upon visiting the main page, I was greeted with just an image — no obvious information. I immediately opened the page source, hoping to find something hidden. There were placeholder characters, which suggested the existence of hidden or backup files like index.php.bak.
Using Wappalyzer, I confirmed the server was running PHP. This encouraged me to try directory brute-forcing with .php and .bak extensions.

Discovering the Source Code
After directory enumeration, I found a backup of the original script: index.php.bak, which contained the full authentication logic.

After reviewing the file, I learned the following:
The server sets two cookies: user and secure_cookie.

The secure_cookie is generated from the string:
$user : $_SERVER['HTTP_USER_AGENT'] : ENC_SECRET_KEY
This string is split into 8-character blocks, and each block is hashed with PHP’s crypt() function using a 2-character salt.

The Vulnerability
I couldn’t simply log in as admin because the username was set automatically in the cookie.
However, I could control the User-Agent header.
That meant I could influence how the string is chunked into 8-byte blocks — and use this to brute-force the key.

My Approach
I decided to brute-force the secret key one character at a time by adjusting the User-Agent length so that the unknown character would fall at the end of the current block.

The algorithm:
I sent a request with a custom-length User-Agent.
I received the secure_cookie, split it into 13-character pieces (each corresponding to a block encrypted by crypt()).
I guessed a character from a payload and computed its hash using the same salt.
When the hash matched the expected block — I added the character to ENC_SECRET_KEY.

Implementation
I wrote a PHP script that:
Uses cURL to send requests.
Extracts cookies from the response.
Dynamically brute-forces the key by comparing block hashes.

Code snippet:
foreach($parts_payload as $p){
    if ($parts_of_scookie[$c_octet_len - 1] == crypt($last7.$p, $hash)){
        echo "found: ". $p."\n";
        $ENC_SECRET_KEY .= $p;
        break; 
    }
}
I had to install the cURL extension for PHP:
sudo apt install php-curl
Result
The brute-force attack worked. I fully recovered the ENC_SECRET_KEY, forged cookies as the admin user, and accessed the protected flag. Challenge completed successfully.
Conclusion
The vulnerability was based on:
The use of outdated PHP crypt() function with a 2-character salt.
The ability to control input via User-Agent.
No use of HMAC or mechanisms to prevent character-by-character brute-force.
This was a textbook example of exploiting a weak cookie-authentication implementation.
