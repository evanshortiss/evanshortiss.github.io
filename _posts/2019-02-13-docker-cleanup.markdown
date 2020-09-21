---
published: true
title: Docker House Keeping
layout: post
categories: development containers docker
redirect_from:
    - /development/containers/docker/2019/02/13/docker-cleanup.html
---

When running large projects that rely on multiple containers I sometimes run into issues with image versions or old images being cached This results in "it works on my machine" issues that a `docker pull` or similar commands usually can't fix.

Sometimes, instead of debugging these issues, I just wipe all running containers and images. A clean slate is nice, right? And, as the internet agrees, "[if it's stupid and it works, it's not stupid](https://knowyourmeme.com/forums/meme-research/topics/34703-if-it-looks-stupid-but-works-it-aint-stupid)".

*WARNING: You will need to download the entire internet again after running this* 😉

```bash
CONTAINERS=$(docker ps -a | tail -n +2 | awk '{print $1}')

echo $CONTAINERS | xargs docker stop
echo $CONTAINERS | xargs docker rm

IMAGES=$(docker images | tail -n +2 | awk '{print $3}')

echo $IMAGES | xargs docker rmi -f
```

You can add this to a function in `.bashrc` or someplace similar to your liking like so:

```bash
function nukedocker {
    echo "Fetching stopped and running containers."
    local containers=$(docker ps -a | tail -n +2 | awk '{print $1}')

    echo "\nStopping containers."
    echo $containers | xargs docker stop

    echo "\nDeleting containers."
    echo $containers | xargs docker rm

    echo "\nFetching images."
    local imageIds=$(docker images | tail -n +2 | awk '{print $3}')

    echo "\nDeleting images."
    echo $imageIds | xargs docker rmi -f
    echo "\nFinished."
}
```
