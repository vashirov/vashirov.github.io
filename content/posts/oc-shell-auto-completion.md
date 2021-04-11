---
title: "How to enable shell auto completion for OpenShift `oc` command"
date: 2021-04-10
draft: false
showtoc: false
tags: ["TIL", "OpenShift", "shell", "zsh", "bash"]
---

I was pleasantly surprised to find out that `oc` (and `kubectl` too) has a built-in capability to generate shell auto completion.

For Zsh add the following snippet to your `~/.zshrc`:

```zsh
if [ $commands[oc] ]; then
    source <(oc completion zsh)
fi
```

For Bash add this to `~/.bashrc`

```bash
if command -v oc &>/dev/null; then
    source <(oc completion bash)
fi
```

After opening a new shell:
```
$ oc <TAB>
adm              describe         logout           replace        
annotate         diff             logs             rollback       
api-resources    edit             new-app          rollout        
api-versions     ex               new-build        rsh            
apply            exec             new-project      rsync          
attach           explain          observe          run            
auth             expose           options          scale          
autoscale        extract          patch            secrets        
cancel-build     get              plugin           serviceaccounts
cluster-info     help             policy           set            
completion       idle             port-forward     start-build    
config           image            process          status         
cp               import-image     project          tag            
create           kustomize        projects         version        
debug            label            proxy            wait           
delete           login            registry         whoami         
```
