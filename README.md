Just some scratch notes for me 
```bash
#local dev 
hugo server #to start local render of webpage 

#simple git commands 
git add. 
git commit -m "xyz"
git push

#if in branch
git status
git add .
git commit -m "branch"
git checkout main
git pull origin main
git merge jwarner
git push origin main
git branch -d jwarner
git push origin --delete jwarner 

# if local is broken
git rm -r --cached themes/blowfish
git submodule add -b main https://github.com/nunocoracao/blowfish.git themes/blowfish
```