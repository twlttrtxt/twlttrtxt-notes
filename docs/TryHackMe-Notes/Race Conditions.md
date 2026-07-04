- Occur when you have limited operations (redeem voucher, send bank money), and multiple threads operate on one single value.

- Exploit by:
1. Intercepting the post request in burp which accesses the resource
2. Sending that to the Repeater
3. Clicking on `+ > New tab group`
4. Add your post request
5. Right click the Post request `> Duplicate Tab` many times
6. Click the arrow next to send, and select `Send group in parallel`, and send it.

- Or use `Battering Ram` attack in intruder