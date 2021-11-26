---
title: Introducing KeeShepherd
permalink: /2021/11/26/introducing-keeshepherd
---
![teaser]({{ site.url }}/images/keeshepherd/teaser2.png)
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

* **track** their positions within a file, so that your config files remain absolutely editable;

* show all your secrets in form of a list, thus giving your a central control point over them:
    <img src="https://user-images.githubusercontent.com/5447190/143605670-e9588533-d649-4467-bd46-dafbcb48a37b.png" width="350px"/>

KeeShepherd is not yet able to detect your secrets automatically, so you'll need to point it to them. That you do by:
* Either **inserting** a secret at the current cursor position:
    <img src="https://user-images.githubusercontent.com/5447190/143601802-2338cb20-946d-4d61-8792-9ff810b974ed.png" width="500px"/>

* Or **adding** the selected secret value into KeyShepherd:
    <img src="https://user-images.githubusercontent.com/5447190/143601857-3dd354c0-5d72-45b5-8103-0e984918aac1.png" width="500px"/>

**Insert** operation by far supports Azure Key Vault, Azure Storage and custom Resource Manager REST API URLs as secret sources, but more sources are on its way.
**Add** operation will suggest to also put the secret value into Azure Key Vault.

Two types of secrets are currently supported:
* **Supervised**. This is a lightweight form of it, just to remember where you left this secret value, let you navigate back to it at any moment and also automatically **mask** them when a config file is opened. Config files themselves are left unchanged.
* **Managed** aka stashable. **Stashing does modifies your config files**, since this is the whole point of it. **Unstashing** restores secret values by fetching them from wherever they're actually stored, e.g. from Azure Key Vault.

It's perfectly fine to mix both **supervised** and **managed** secrets in the same config file. A good strategy could be to mark real secrets (access keys, connection strings etc.) as managed (to keep them safe) and leave less important values like user names, application ids etc. as supervised (to make it easy to find them later).

KeeShepherd can also do **automatic stashing/unstashing** once you open/close a workspace. Default mode is to automatically stash and do not automatically unstash, but you can configure this via Settings:
![image](https://user-images.githubusercontent.com/5447190/143603144-1a003f0c-7173-4225-8fa4-102cef545a4a.png)
Automatic **stashing/unstashing** seems to be the most secure form, it ensures that your secret values are only present in the config files while you're actually working on the project (aka while a VsCode window is open). But this mechanism is, of course, not immune to VsCode accidental crashes, so better to explicitly check whether it all went as planned.

As I said before, KeyShepherd does not store your secret values. In its metadata store it only stores salted SHA-256 hashes of them and their "coordinates" aka file paths and last known positions. This metadata store can be in form of:
* Local JSON-files in VsCode's global storage folder (`C:\Users\user-name\AppData\Roaming\Code\User\globalStorage\kee-shepherd.kee-shepherd-vscode` on Windows).
* Shared Azure Table. This mode gives you a bird's eye view of all your (and your teammate's) secrets on all your devboxes, including [GitHub Codespaces](https://github.com/features/codespaces) (yes, KeeShepherd extension works there as well).

KeeShepherd will ask you what metadata storage type you prefer at first run, but you can always change it later with `Switch to Another Metadata Storage` command:
    <img src="https://user-images.githubusercontent.com/5447190/143605899-d3178d84-a126-41b4-a74e-4eb74a2b7d16.png" width="350px"/>



All of this is only the beginning, of course. Many other new features are to be added in the next version(s), for example:
* More secret sources, like Azure Service Bus, Azure Event Hubs, Azure SQL etc. etc.
* Prevent accidentally committing unstashed secret values (probably, by temporarily adding their file names to .gitignore).
* Automatically resolve secrets by names, even in a different file on a different machine.
* Easily rotate (modify) secret values from 'SECRETS' view.
* ... [<put your suggestions here>](https://github.com/scale-tone/kee-shepherd/issues) ...
    
Other incarnations of KeeShepherd, e.g. CLI version and/or plugins for other IDEs are also very much on the radar.
But so far you're very welcome to [install it from VsCode marketplace](https://marketplace.visualstudio.com/items?itemName=kee-shepherd.kee-shepherd-vscode) and enjoy.
