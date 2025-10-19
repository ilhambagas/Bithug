# Bithug
This is a difficult web exploitation task. To solve it, I must exploit a Server-Side Request Forgery (SSRF) vulnerability in Git webhooks and use template injection. This allowed me to gain admin privileges via localhost, create a custom Git packfile to add myself as a collaborator to the hidden repository, and then clone it to read the flag.
