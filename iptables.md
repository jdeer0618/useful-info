##iptables commands

####view rules with log lines and no resolution
`iptables -nL --line-numbers`

####insert rule at rule number
`iptables -I INPUT 1 -p tcp --dport 9013 -j ACCEPT`

####save rules and make persistant
`service iptables save`
