# blog for myself

## setup locally

```bash
gh repo clone vincentbockaert/me.vincentbockaert.xyz
cd me.vincentbockaert.xyz
git submodule update --init --recursive
hugo server # add flag -D for enabling draft mode
```

## publishing

simply run the below command and the static content will be generated into a `public` folder

```bash
hugo
```
