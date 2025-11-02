# Initial setup

## Git/GitHub Integration

This folder probably wont contain much. However, I wanted to document any git/github commands that are useful for server usage. 

### gh

GitHub provides a suite of command line tools that are useful for working with remote repositories on a server.

`sudo apt install gh`

Most commonly, simply initializing a github repository and connecting it to the server can be done via cli. For example, for this repo, I used:

```
gh auth login (only needed to add auth to the server)
gh repo create linux-lab --public --clone
cd linux-lab
mkdir -p 01-core-infra/docs
touch README.md
```



