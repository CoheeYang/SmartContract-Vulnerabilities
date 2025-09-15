# Inflation Attack

inflation attack主要是攻击那些新建的ERC4626 Vault。

这些Vault通过将mint shares来给token存放人凭证，并按share在`totalsupply`的占比来给持有人对应的赎回额。

Inflation Attack是在vault还没有存入的情况下，当有人存入100 token时，抢跑1 token 获得 1shares(100%)，再通过捐赠100token扩大1share所对应的token价值，此时是1share = 101token，之后被害者100token存入，由于它存入的100 token，理论获得100/101个shares，但是由于向下取整为0 shares。

攻击者此时withdraw所有token，白嫖100token。

Mitigation:

https://blog.openzeppelin.com/a-novel-defense-against-erc4626-inflation-attacks