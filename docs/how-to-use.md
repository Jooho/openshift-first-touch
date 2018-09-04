How to use OpenShift First Touch Ansible
----------------------------------------

Each branch has separate ansible code based on OpenShift version.

For example, if you are using OpenShift 3.9, you have to use branch 3.9. 

As for variable, you can find default variable from *./vars/default.yml*. If you want to override default variable, please use “./vars/override.yml” file. One thing you have to check is the latest version of image. Do **NOT** change the default values from *./vars/defaults.yml



## Clone git repository
```
git clone https://github.com/Jooho/openshift-first-touch.git

cd openshift-first-touch
```

## Check Branchs
```
 git branch -a
* master
  remotes/origin/3.7
  remotes/origin/3.9
  remotes/origin/HEAD -> origin/master
  remotes/origin/master
```

## Change branch
```
# Example OCP 3.7
$ git checkout -b 3.7 origin/3.7  

# Example OCP 3.9
$ git checkout -b 3.9 origin/3.9 
```

## Check default variables
```
$ cat ./vars/default.yml
```

## Update variables
```
$ vi ./vars/override.yml
```


