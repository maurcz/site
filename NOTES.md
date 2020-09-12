Future me will love this file.

## Hugo

- `hugo -D server` to run local server (drafts enabled).
- `hugo new posts/NNN-your-new-post.md` where `NNN` should be a 3 digit number. 
- Static content should go under `static/**/NNN/file.extension`. 

## Deploy

From https://gohugo.io/hosting-and-deployment/hosting-on-github/:

- Run `hugo` to generate public html files.
- Run `./deploy.sh "Your optional commit message"` to send changes to `<USERNAME>.github.io`. 
- `commit` and `push` changes to this repo.
