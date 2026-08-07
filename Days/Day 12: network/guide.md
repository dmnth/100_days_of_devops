1. Determine what server is failing 

curl http://stapp01:8082 - unable to respond

2. Install netstat 

sudo dnf install netstat

3. Check what is blocking httpd from startup

sudo netstat -tulpn

4. Kill the process, restart httpd

sudo systemctl restart httpd

5. Verify whether httpd running

sudo systemctl status httpd

6. From remote host fail to reach the web-server

7. Back to stapp01 to check firewall rules and find one that blocking all traffic:

6   360 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited

8. Add exception to allow incoming tcp traffic on port 8082

sudo iptables -I INPUT -p tcp --dport 8082 -j ACCEPT

9. Confirm it landed above REJECT:

sudo iptables -L INPUT -n -v --line-numbers

10. Save changes:

sudo iptables-save

11. From remote host verify whether http://stapp01:8082 is no reachable with curl
