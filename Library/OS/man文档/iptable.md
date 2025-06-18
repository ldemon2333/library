A firewall rule specifies criteria for a packet and a target. If the packet does not match, the next rule in the chain is examined; if it does match, then the next rule is specified by the value of the target, which can be the name of a user-defined chain.

ACCEPT means to let the packet through.  DROP means to drop the packet  on  the  floor.
RETURN  means  stop  traversing  this chain and resume at the next rule in the previous
(calling) chain.  If the end of a built-in chain is reached or a  rule  in  a  built-in
chain  with  target  RETURN is matched, the target specified by the chain policy deter‐
mines the fate of the packet.

1