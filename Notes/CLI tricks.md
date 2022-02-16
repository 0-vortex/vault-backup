# GH CLI tricks

## Basic iteration loop

```shell
gh repo list Tiamat-Tech --limit 99999 |
    while read -r repo _; do 
        if [[ $repo =~ "^(Tiamat-Tech/*)" ]]; then 
            echo "$repo"; 
        fi;     
        sleep 0.1; 
    done
```

## Quick no-sync backup

```shell
gh repo list Tiamat-Tech --limit 99999 |
    while read -r repo _; do 
        if [[ $repo =~ "^(Tiamat-Tech/*)" ]]; then 
            echo "$repo"; 
        fi;     
        sleep 0.1; 
    done
```