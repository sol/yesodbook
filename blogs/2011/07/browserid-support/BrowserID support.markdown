I just implemented [BrowserID](http://browserid.org/) support for [Haskellers](http://www.haskellers.com/), and wanted to share some quick thoughts. Firstly, the process is a piece of cake: it amounted to about one line of HTML, 4 lines of Javascript, and 10 lines of Haskell. (The commit to the repo is significantly larger, since there is some logic for backwards-compatibility with verified emails, but that's a separate issue.)

I think this system is much more user friendly than OpenID. You get an identifier that you can actually remember (as opposed to Google's OpenID implementation). There's less of a question of which identifier you used. With OpenID, there's always the question of whether you logged in with Google, Wordpress, or Yahoo. Now the only question is which email address you used, which I would imagine is a simpler one to track. Also, BrowserID's handling of multiple email addresses is very nice for people who share computers with spouses/roommates.

## User testing

I happened to have two willing test subjects (my sisters) visiting me, and I asked them each to log in with BrowserID. Here are the issues they ran up against:

* They were very concerned giving their email address to a site they didn't recognize.
* The initial login window was unclear, asking for email and password. This implied that she was supposed to provide the password for her email account, which made her even more concerned.
* Since my sister uses a webmail based account, she verified by going back to the original browser window and opening a new tab to her email account. Once she verified, she went back to the Haskellers tab. However, since the BrowserID window was already open, the "Login" button was no longer responsive.

It seems like these issues could be overcome with integration within the web browser and user education.

## Yesod integration

So far, all of the BrowserID code is in Haskellers itself. After some testing, I'm planning on adding support to both the authenticate and yesod-auth packages. After a little more testing, I will likely include BrowserID in the default scaffolded site. If things go as well as they seem to be, I'd likely use BrowserID as the primary (or perhaps only) login method for future sites.