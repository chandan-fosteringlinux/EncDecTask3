# **Setting Up Apache Server for Symmetric Encryption and Decryption**

## **Prerequisites**

* **Operating System**: Ubuntu 20.04 LTS or later.  
* **Sudo Privileges**: Ensure you have the necessary permissions to install and configure packages.  
* **Required Packages**: OpenSSL must be installed; check with `openssl version`.  
* **Installed Apache Server**: Ensure Apache is running before proceeding with the setup.

## **1\. Install and Start Apache Server**

Ensure that Apache is installed and running on your system. Execute the following commands:

`sudo apt update`  
`sudo apt install apache2`  
`sudo systemctl start apache2`  
`sudo systemctl enable apache2`

## **2\. Enable Required Apache Modules**

You'll need to enable specific Apache modules to handle custom HTTP headers and server-side scripts:

`sudo a2enmod headers`  
`sudo a2enmod cgi`  
`sudo systemctl restart apache2`

## **3\. Generate a Symmetric Key**

Create a symmetric key for encryption:

`openssl rand -base64 32 > symmetric.key`

## **4\. Encrypt the Message**

Create a message.txt file using the command  
echo "this is my secret message" \>\> message.txt

Using the symmetric key, encrypt your message file:

`openssl enc -aes-256-cbc -salt -in message.txt -out encrypted_message.enc -pass file:./symmetric.key`

## **5\. Prepare the Encrypted File Directory**

Create a directory to store the encrypted file:

`sudo mkdir /var/www/html/encrypted`

Copy the encrypted message to the directory:

`sudo cp encrypted_message.enc /var/www/html/encrypted/`

## **6\. Set Permissions for the Encrypted Folder**

Change the ownership of the encrypted folder to the `www-data` user and group:

`sudo chown -R www-data:www-data /var/www/html/encrypted`

## **7\. Create a Bash Script for Decryption**

Create a script to handle custom headers from the browser:

`sudo nano /usr/lib/cgi-bin/decrypt.sh`

### **Add the Following Script to `decrypt.sh`**

`#!/usr/bin/bash`

`# Output Content-Type header`  
`echo "Content-type: text/plain"`  
`echo ""`

`# Parse the X-Symmetric-Key from the query string`  
`DECRYPT_KEY=$(echo "$QUERY_STRING" | grep -oP 'X-Symmetric-Key=\K[^&]*')`

`# URL decode the key (if needed)`  
`DECRYPT_KEY=$(echo -e "$(echo "$DECRYPT_KEY" | sed 's/+/ /g;s/%\(..\)/\\x\1/g;')")`

`# Define the encrypted file path`  
`ENCRYPTED_FILE="/var/www/html/encrypted/encrypted_message.enc"`  
`Encrypted_content=$(cat "$ENCRYPTED_FILE")`

`# Check if the decryption key is provided`  
`if [ -z "$DECRYPT_KEY" ]; then`  
  `echo "No decryption key provided."`  
  `echo "$Encrypted_content"`   
  `exit 1`  
`fi`

`# Decrypt the file using the provided key`  
`DECRYPTED_CONTENT=$(openssl enc -aes-256-cbc -d -in "$ENCRYPTED_FILE" -pass pass:"$DECRYPT_KEY" 2>&1)`

`# Check if decryption was successful`  
`if [[ "$DECRYPTED_CONTENT" == *"bad decrypt"* ]]; then`  
  `echo "Invalid decryption key."`  
  `echo "$Encrypted_content"`  
  `exit 1`  
`fi`

`# Output the decrypted content`  
`echo "$DECRYPTED_CONTENT"`

## **8\. Set Executable Permissions for the Script**

Make the script executable:

`sudo chmod +x /usr/lib/cgi-bin/decrypt.sh`

## **9\. Configure Apache to Use the Script**

Edit the Apache configuration file to set up the script alias and permissions:

`sudo nano /etc/apache2/sites-available/000-default.conf`

### **Add the Following Configuration inside \<VirtualHost \*:80\> block**

`ScriptAlias /cgi-bin/ /usr/lib/cgi-bin/`  
`<Directory "/usr/lib/cgi-bin">`  
      `AllowOverride None`  
      `Options +ExecCGI`  
      `Require all granted`  
`</Directory>`  
`AddHandler cgi-script .cgi .sh`

## **10\. Restart Apache Server**

Restart the Apache server to apply the changes:

`sudo systemctl restart apache2`

## **11\. Access the Script in the Browser**

Navigate to the script in your MozilaFireFox browser by entering the below url address followed by the script path. For example:

`localhost/cgi-bin/decrypt.sh`

##   **12\. Test the Decryption Script**

1. Open the developer tools in Firefox browser (Ctrl \+ Shift \+ I).  
2. Refresh the page and find the GET request for `decrypt.sh` in the network section.  
3. Right-click on the request and select "Edit and Resend."  
4. In the URL parameter section, add the custom header:  
   * Name: `X-Symmetric-Key`  
   * Value: `<Your_Symmetric_Key>` (e.g., `RDDQ/W/bSgC8xguNSIilKXkdrUwzUHElArGVXbR0Uls=`)  
5. Press the **Send** button after providing the custom headers. This will generate the URL with the custom headers.  
6. Copy the generated GET URL with the custom headers.  
7. Paste that URL into the browser's address bar and press **Enter**.  
 
