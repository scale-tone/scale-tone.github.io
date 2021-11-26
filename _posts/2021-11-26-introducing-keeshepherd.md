---
title: Introducing KeeShepherd
permalink: /2021/11/26/introducing-keeshepherd
---
![teaser]({{ site.url }}/images/keeshepherd/teaser1.png)
# Introducing KeeShepherd.



*[Quick link to VsCode marketplace](https://marketplace.visualstudio.com/items?itemName=kee-shepherd.kee-shepherd-vscode)*

*[Quick link to the repo](https://github.com/scale-tone/kee-shepherd)*

I hate placing credentials (connection strings, access keys, secrets, passwords etc.) into config files (`web.config`, `app.config`, `.env`, `local.settings.json` etc.). Every time I do that I get nervous, because:
* I always forget where I left them. So they quickly get scattered across countless folders and devboxes, like lobsters crawling out of a trap. Difficult to rotate and difficult to cleanup.
* I always face the fear of accidentally exposing them during demo sessions. Unlike password fields in browsers, config files are just a text, and IDEs are not yet smart enough to automatically hide secrets in them.
* I'm always afraid of accidentally committing them to the repo. `.gitignore` certainly helps, but I still need to remember to configure it properly.

Yes, it seems like we're [heading towards the bright passwordless future](https://www.microsoft.com/security/blog/2021/09/15/the-passwordless-future-is-here-for-your-microsoft-account/), but for us developers working with various cloud services and databases that future hasn't arrived yet.

So as a temporary solution for the next decade or so I created [KeeShepherd](https://github.com/scale-tone/kee-shepherd/tree/main/kee-shepherd-vscode#readme). This tool comes [in form of a VsCode extension](https://marketplace.visualstudio.com/items?itemName=kee-shepherd.kee-shepherd-vscode), and **no, it is not a password manager**. 
It does not store secret values, only cryptographically strong salted SHA-256 hashes of them, links to them (whether it is a link to an Azure Key Vault secret or to a secret accessible via Azure Resource Manager REST API, like [Storage access keys](https://docs.microsoft.com/en-us/rest/api/storagerp/storage-accounts/list-keys)), their lengths and positions in config files. Yet this information is enough for KeeShepherd to be able to:
* automatically **mask** (hide) secret values once you open a config file:
    ![image](https://user-images.githubusercontent.com/5447190/143591791-8e59a1bd-9448-441c-b99b-301da04c73f2.png)

* (automatically) **stash** (aka replace with anchors like `@KeeShepherd(secret-name)`) and **unstash** (aka do the opposite) them:
    ![image](https://user-images.githubusercontent.com/5447190/143592201-94911fa8-b651-44b6-8692-4a6b2749d5a7.png)    
    
* show all your secrets in form of a list, thus giving your a central control point over them:
    ![image](https://user-images.githubusercontent.com/5447190/143592931-389e1f44-e8d9-40b8-b581-1f1ded51eb0b.png)
