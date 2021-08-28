---
title: "Keeping Terraform module sources up-to-date"
date: 2021-08-28T13:41:25-06:00
draft: false
---

Few weeks ago, I decided to write a tool called [drdrei](https://github.com/fmterrorf/drdrei) that will automatically check if the module's source attribute isn't using the latest version. This tool only works if you have a mono repo that uses tag name format such as `<feature_name>-<semver>` for example: 

```
	gcs-1.0.1
	gcs-1.0.2
	gcs-1.0.3
```

And in you TF you might have the following:

```hcl
module "blahblah" { 
  source = "git::https://example.com/monorepo.git?ref=gcs-v1.0.1"
}
```

You can run `drdrei path/to/module` and it will tell you to use use `gcs-1.0.3`

## How it works

Building **drdrei** is pretty simple since all i have to do use existing go libraries & git features. Folks at Terraform are generous enough to make this helper tool called [terraform-config-inspect](https://github.com/hashicorp/terraform-config-inspect) that i used to get high-level metadata such as the `source` attribute) that i used to get high-level metadata such as the `source` attribute from the module. I then used regex to parse out the feature name & semver info from the source.

Once i have the sources, i can `git remote-ls` to fetch the list of tags from the source git repo. I then sort the tags using [Masterminds/semver](https://github.com/Masterminds/semver) & only fetch the latest one. Then i compare if my local copy is using the the latest version & show output to the console if it isn't.

## Conclusion

I had fun building drdrei. It was simple, didn't take too much time to build & does the job. One of my coworkers suggested to put in our pre-commit hooks & you can use the script below.

```sh
module_dirs=$(git diff --cached --name-only --diff-filter=d "*.tf" \
	| xargs -n1 dirname \
	| tr '\n' ' ')

[ -n "${module_dirs}" ]\
	&& echo "Auditing outdated module sources"\
	&& drdrei $module_dirs
```

