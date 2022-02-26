[Source](https://coderwall.com/p/euwpig/a-better-git-log)

This command will add an alias of `lg` to  `git` for showing better logs:

`git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"`
