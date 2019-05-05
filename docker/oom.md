| Hi there,We'd like to add a new feature to Docker say, disable oom killer when OOM happend[\#11034](https://github.com/moby/moby/pull/11034). This is useful especially in production scenario but we have not yet been in accord on what's the name would this option be.By now,`--disable-oom-kill`is in lead but the name is a little too long, WDYT. So if someone else has a strong preference, please comment. |
| :--- |


[![](https://avatars1.githubusercontent.com/u/1826947?v=3&s=88 "@phemmer")](https://github.com/phemmer)

Contributor

### [**phemmer**](https://github.com/phemmer)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94347919)

| Instead of a boolean to disable the oom killer entirely \(which is a really bad idea in my opinion\), how about a tunable instead? Meaning something like`--oom-score=100`. This would use the standard linux`oom_score_adj`control mechanism. The score controls the tendency for the kernel to target that process when it needs to kill something, and a score of`-1000`means to disable the OOM killer for that process. |
| :--- |




### [**thockin**](https://github.com/thockin)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94349344)

| This is heading in the direction of what we call Priority in Borg. OOM scoring is just one facet of it, but an important one.On Sun, Apr 19, 2015 at 8:35 PM, Patrick Hemmer[notifications@github.com](mailto:notifications@github.com) wrote:Instead of a boolean to disable the oom killer entirely \(which is a really bad idea in my opinion\), how about a tunable instead? Meaning something like --oom-score=100. This would use the standard linux oom\_score\_adj control mechanism. The score controls the tendency for the kernel to target that process when it needs to kill something, and a score of -1000 means to disable the OOM killer for that process.— Reply to this email directly or view it on GitHub [docker\#12528 \(comment\)](https://github.com/moby/moby/issues/12528#issuecomment-94347919). |
| :--- |


[![](https://avatars3.githubusercontent.com/u/9958621?v=3&s=88 "@HuKeping")](https://github.com/HuKeping)

Contributor

### [**HuKeping**](https://github.com/HuKeping)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94349457)

| Hi[@phemmer](https://github.com/phemmer), the oom\_score\_adj is related to a pid and we can't get that before the container start. |
| :--- |


[![](https://avatars2.githubusercontent.com/u/101445?v=3&s=88 "@LK4D4")](https://github.com/LK4D4)

Contributor

### [**LK4D4**](https://github.com/LK4D4)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94352041)

| I agree with[@HuKeping](https://github.com/hukeping)that oom\_control and oom\_score\_adj have different targets - cgroup and process. So I think it should be two different options. |
| :--- |


[![](https://avatars1.githubusercontent.com/u/1826947?v=3&s=88 "@phemmer")](https://github.com/phemmer)

Contributor

### [**phemmer**](https://github.com/phemmer)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94354377)

| I agree with[@HuKeping](https://github.com/hukeping)that oom\_control and oom\_score\_adj have different targets - cgroup and process.I'm not disagreeing, but`oom_score_adj`offers a lot more flexibility, and can do what`oom_control`does \(in this scenario\). |
| :--- |


[![](https://avatars1.githubusercontent.com/u/1826947?v=3&s=88 "@phemmer")](https://github.com/phemmer)

Contributor

### [**phemmer**](https://github.com/phemmer)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94358589)

| Here's a PoC showing it working \(score hard coded to`123`\): [phemmer/docker@3fe3a89](https://github.com/phemmer/docker/commit/3fe3a89a1d04222b9f3308283b564cb53c2b311c)\# docker run --rm -ti busybox sh -c 'cat /proc/self/oom\_score\_adj'123 |
| :--- |


[![](https://avatars3.githubusercontent.com/u/9958621?v=3&s=88 "@HuKeping")](https://github.com/HuKeping)

Contributor

### [**HuKeping**](https://github.com/HuKeping)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94370445)

| hi[@phemmer](https://github.com/phemmer), When OOM happen, the kernel will try to find out who was the most likely to be killed and that was depend on the oom\_score. Generally, the highest score owner will be killed, then how to weight when we doing the creating things? Why should you assign container\_1 with 100 and another with 200?I still think we use the cgroup to control that is safe and enough. |
| :--- |


[![](https://avatars1.githubusercontent.com/u/1826947?v=3&s=88 "@phemmer")](https://github.com/phemmer)

Contributor

### [**phemmer**](https://github.com/phemmer)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94372316)

| Why should you assign container\_1 with 100 and another with 200?If`container_1`is your database container, and`another`is your application container, your database container will be less likely to be targeted by the OOM killer. This is the behavior you want as killing your database container can result in DB corruption, where as killing the application container is likely to be no more than an inconvenience.Disabling the OOM killer is a very bad idea as this leaves the kernel with very minimal choices of what it can kill. If you've got a server that runs nothing but docker \(and containers started by docker\), and you have the OOM killer disabled on every container, the OOM killer is likely to target docker itself, or other critical system processes \(network manager, dbus-daemon, etc\), of which killing them can lead to major system instability. If you're going to disable the OOM killer, you should be doing it by disabling overcommit. |
| :--- |


[![](https://avatars3.githubusercontent.com/u/9958621?v=3&s=88 "@HuKeping")](https://github.com/HuKeping)

Contributor

### [**HuKeping**](https://github.com/HuKeping)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94379025)

| Docker is not a toy, in the real world, in every cloud platform , the provider will always run docker for their customers with option`-m`and this will~~prevent system memory leaking.~~ On the customer side, they would prefer not kill their app directly when OOM happened, that is too aggressive to acceptable.EDIT: I mean with the option`-m`, even if we disable the oom killer ,it will not hang the whole system. |
| :--- |


[![](https://avatars1.githubusercontent.com/u/1826947?v=3&s=88 "@phemmer")](https://github.com/phemmer)

Contributor

### [**phemmer**](https://github.com/phemmer)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94439471)

| I think some of the confusion might be coming in on what the OOM killer is for. The OOM killer is turned on when the system has overcommit enabled. Overcommit is basically when the kernel allows programs to allocate more memory than the system actually has. It does this in the hope that the programs on the host won't actually use all the memory they allocate. For example, an application allocates a 1mb buffer to read some data from the network, but it only ends up reading a few kb. In this scenario you have several hundred kb of unused memory. Overcommit prevents wasting this memory. Now lets say the application ends up using that several hundred kb of extra memory. The kernel already granted the memory to the application, so the application isn't performing a`malloc`call which the kernel can refuse. So the kernel is obligated to give the program the memory it said it has. This is where the OOM killer comes in. In this scenario, it invokes the OOM killer to kill a program and get some memory back.As mentioned, if you disable the OOM killer for all your containers, the only thing left which the OOM killer can kill is stuff outside the containers. If you run a box dedicated to docker, the only stuff outside containers is likely to be critical system processes. Killing these processes can result in major system instability.Docker is not a toy, in the real world, in every cloud platform , the provider will always run docker for their customers with option -m and this will prevent system memory leaking.This has absolutely no impact on the scenario. The cgroup memory limiting features of the kernel do not disable overcommit. If you limit a container to 100m, and the program performs a 1gb`malloc`call, the call will not be refused.On the customer side, they would prefer not kill their app directly when OOM happened, that is too aggressive to acceptable.While yes, customers would not like their programs getting killed, any hosting provider would prefer to lose a single container rather than have the entire box go down.As I mentioned earlier, the only proper solution to disabling the OOM killer is to disable memory overcommit. With overcommit disabled, the kernel will instead refuse`malloc`calls if the`malloc`would use more memory than allowed by the cgroup limit, or is available on the box. The OOM killer isn't even enabled when overcommit is disabled, as the scenario where the kernel gives out more memory than the box has can never happen. |
| :--- |


[![](https://avatars3.githubusercontent.com/u/9958621?v=3&s=88 "@HuKeping")](https://github.com/HuKeping)

Contributor

### [**HuKeping**](https://github.com/HuKeping)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94446183)

| While yes, customers would not like their programs getting killed, any hosting provider would prefer to lose a single container rather than have the entire box go down.how can this happen if you limit the container with`-m`, so you will give a tons of memory to it? |
| :--- |


[![](https://avatars0.githubusercontent.com/u/6415670?v=3&s=88 "@hqhq")](https://github.com/hqhq)

Contributor

### [**hqhq**](https://github.com/hqhq)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94459369)

| [@phemmer](https://github.com/phemmer)One scenario is that user don't want his app be killed but just be hung when the container mistakenly consume more memory than expected. That has nothing to do with system hung or overcommit usage. So oom kill disable is useful.Oom score support is useful too, but it can't cover all scenarios of oom kill disable does, like the above scenario.[@HuKeping](https://github.com/hukeping)You don't get[@phemmer](https://github.com/phemmer)'s point, even you limit container with -m, if you overcommit, there is still chance to crash the system. Think about you have 10 containers each of them has a limit of 500M, but you have only 4G memory on the host, if you disable oom kill for all containers, when host got low memory, the only choice it has is to kill some processes outside of containers which might be much more important. |
| :--- |


[![](https://avatars1.githubusercontent.com/u/1826947?v=3&s=88 "@phemmer")](https://github.com/phemmer)

Contributor

### [**phemmer**](https://github.com/phemmer)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94470287)

| how can this happen if you limit the container with -m, so you will give a tons of memory to it?As I mentioned:The cgroup memory limiting features of the kernel do not disable overcommit. If you limit a container to 100m, and the program performs a 1gb malloc call, the call will not be refused.To make this point, try the following:docker run -ti -m 1g gcc sh -c 'cd /tmp && curl [https://gist.githubusercontent.com/phemmer/fff60f857ee8ca5a74cd/raw/044f8f59be0b21857ee310e030a9629517fdfeb4/gistfile1.txt](https://gist.githubusercontent.com/phemmer/fff60f857ee8ca5a74cd/raw/044f8f59be0b21857ee310e030a9629517fdfeb4/gistfile1.txt) &gt; test.c && make test && ./test'This command will compile and run an application which allocates 2gb of memory. It runs perfectly on any system which allows overcommit \(notice that the docker container is run with`-m 1g`\).[@phemmer](https://github.com/phemmer)One scenario is that user don't want his app be killed but just be hung when the container mistakenly consume more memory than expected. That has nothing to do with system hung or overcommit usage. So oom kill disable is usable.But this isn't what happens. If the box runs out of memory, the kernel isn't going to just hang the app, it's going to kill something. As long as the OOM killer is enabled, it's going to find something to kill, and it's going to keep killing until it either gets the memory it needs, or the kernel panics.And as much as I consider disabling OOM killer as bad, my proposal does not preclude the possiblity of disabling the OOM killer. My proposal is about using a tunable instead of a binary on/off. It would be a lot more useful to be able to say "container X should be less likely to be targeted by OOM than container Y". As is covered in the`oom_score_adj`documentation, a score of`-1000`will effectively disable the OOM killer. |
| :--- |


[![](https://avatars1.githubusercontent.com/u/5595220?v=3&s=88 "@thockin")](https://github.com/thockin)

Contributor

### [**thockin**](https://github.com/thockin)commented[on Apr 20 2015](https://github.com/moby/moby/issues/12528#issuecomment-94488388)

| [@HuKeping](https://github.com/hukeping)how can this happen if you limit the container with -m, so you will give a tons of memory to it?There are two grades of OOM - "local" memcg OOM and system OOM. If your container uses more memory than you asked for, you get a local OOM and you should be killed.If the system overall is overcommitted, SOMEONE has to die, though you don't know who. This is what OOM scoring is for |
| :--- |


[![](https://avatars0.githubusercontent.com/u/6415670?v=3&s=88 "@hqhq")](https://github.com/hqhq)

Contributor

### [**hqhq**](https://github.com/hqhq)commented[on Apr 21 2015](https://github.com/moby/moby/issues/12528#issuecomment-94607877)

| But this isn't what happens. If the box runs out of memory, the kernel isn't going to just hang the app, it's going to kill something. As long as the OOM killer is enabled, it's going to find something to kill, and it's going to keep killing until it either gets the memory it needs, or the kernel panics.I'm talking about disabling cgroup oom, which is what this propose tried to do. If you disable a cgroup's oom, when the cgroup runs out of memory, it will hung the process which was trying to get some memory and can't, put it to wait queue until memory is available. Nothing will be killed. |
| :--- |


### ![](https://avatars2.githubusercontent.com/u/9958621?v=3&s=32)[HuKeping](https://github.com/HuKeping)closed this[on May 5 2015](https://github.com/moby/moby/issues/12528#event-296789899) {#event-296789899}

### This was referencedon May 11 2015

Closed

#### [Support setting OOM score\#13116](https://github.com/moby/moby/issues/13116)

Closed

#### [oom\_kill\_disable poorly understood & implemented\#14440](https://github.com/moby/moby/issues/14440)

[![](https://avatars0.githubusercontent.com/u/640797?v=3&s=88 "@CMCDragonkai")](https://github.com/CMCDragonkai)

### [**CMCDragonkai**](https://github.com/CMCDragonkai)commented[on Dec 15 2015](https://github.com/moby/moby/issues/12528#issuecomment-164801816)

| [@phemmer](https://github.com/phemmer)I'm curious about whether these 2 situations are possible:First that the host system has overcommit enabled but inside the container overcommit is disabled where malloc will fail once past the mem limit. No memcg OOM is possible here, but the host OOM is still possible if the user is overcommitting containers.Second is where the host system has overcommit disabled, but inside the container, overcommit is enabled. This means the process malloc succeeds past the mem limit but will get OOM killed. But the malloc can also fail in general if the container being created is going past the OS total memory. |
| :--- |


[![](https://avatars1.githubusercontent.com/u/1826947?v=3&s=88 "@phemmer")](https://github.com/phemmer)

Contributor

### [**phemmer**](https://github.com/phemmer)commented[on Dec 16 2015](https://github.com/moby/moby/issues/12528#issuecomment-164997117)

| First that the host system has overcommit enabled but inside the container overcommit is disabled where malloc will fail once past the mem limit. No memcg OOM is possible here, but the host OOM is still possible if the user is overcommitting containers.Well, you certainly can't do it via the normal sysctl/`/proc/sys`controls. I just tried, and killed my host when I set it inside the container. Oops. I just looked to see if there was a way to do this via cgroups, and ran across[this patch proposal](http://lists.linuxfoundation.org/pipermail/containers/2008-June/011202.html)which[was rejected](http://lists.linuxfoundation.org/pipermail/containers/2008-June/011208.html)because another feature in the kernel could be used to accomplish the same goal, memrlimit. However it appears either memrlimit was removed from the kernel, or never made it past testing, as current kernels have no mention of it. TL;DR: does not appear possible.Second is where the host system has overcommit disabled, but inside the container, overcommit is enabled. This means the process malloc succeeds past the mem limit but will get OOM killed. But the malloc can also fail in general if the container being created is going past the OS total memory.Given that you can't have separate overcommit settings inside the container, and outside, this scenario isn't possible. |
| :--- |




