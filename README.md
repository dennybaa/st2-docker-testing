
# Docker MultiContainer Layout

## Files are written using puppet and all is orchestrated with too much code and tools (including packer, puppet...). <u>Complexity,  Difficult!</u>

**Too many** tools to support containers like puppet, ruby, tiller, packer and maybe smth else. `Tiller` is good, but better to find smth pythonish (if it's finally needed) - just not making it necessary to bring ruby inside.

## Client container is missing

No container, thus requires D.I.Y to play with stackstorm containers bunch. See manual testing.


## Manual testing of actions in multicontainer (I haven't tried sensors and webhooks)

Get the internal IP of st2auth container (bugs of st2auth container should be fixed before, otherwise won't start).

After this we can login into actionrunner `docker exec -it st2dockertesting_st2actionrunner_1 /bin/bash` and try issuing commands:

```
export TERM=xterm
export ST2_AUTH_URL="http://!PUT_YOUR_AUTH_IP!:9100"
export ST2_AUTH_TOKEN=$(st2 auth -p testp testu | grep token | cut -f3 -d\| | sed -r 's/\s+//')

cd /opt/stackstorm/src
source virtualenv/bin/activate
export PYTHONPATH=$PYTHONPATH:/opt/stackstorm/src/st2common:/opt/stackstorm/src/st2reactor:/opt/stackstorm/src/st2actions:/opt/stackstorm/src/st2api:/opt/stackstorm/src/st2auth:/opt/stackstorm/src/st2debug
python st2common/bin/st2-register-content --config-file /etc/st2/st2.conf --register-actions

# TEST ACTIONS
st2 run core.local -- hostname
st2 execution get !YOUR_EXEC_ID!
```

If the above has been well, so than at list api, actionrunner, resultracker connections to mongo and redis are okay.

### Result

Playing around with containers built with packer, st2factory, puppet and etc... Most of the things seem to operate, the bugs bellow should be fixed however.

Personally, I can say supporting these containers is not fun, seems that there'll be no folks who might want to use them, this is my opinion anyway.

## Bugs

1.  Puppet & packer container build process doesn't populate  `httpasswd` file for **st2auth** container.
  ```
st2auth_1            |   File "/opt/stackstorm/src/virtualenv/local/lib/python2.7/site-packages/passlib/apache.py", line 234, in load
st2auth_1            |     with open(self._path, "rb") as fh:
st2auth_1            | IOError: [Errno 2] No such file or directory: u'/etc/st2/htpasswd'
  ```
1. No stanley on actionrunner (*if it's supposed that action runners can run actions locally*). I guess other containers might need stanley as well.
  ```
{
    "failed": true, 
    "stderr": "sudo: unknown user: stanley
sudo: unable to initialize policy plugin
", 
    "return_code": 1, 
    "succeeded": false, 
    "stdout": ""
}
  ```
1. Another one I've made PR https://github.com/StackStorm/puppet-st2/pull/20 for.

## Proper container layout docker-way

- We don't need or want  base ("raw") images build with packer and other magical tools chef, puppet etc.
- Docker container should be built with Dockerfile.
  - Should provide **clarity** - https://github.com/docker-library/official-images#clarity
  - Should provide **cacheability** - https://github.com/docker-library/official-images#cacheability
  - etc.
   
   We need to make **simple and clear** Dockerfile for stackstorm containers. It's better to resemble the way how other containers are built, for example - https://github.com/docker-library/rabbitmq/blob/master/Dockerfile. And we should follow principles of official docker images https://github.com/docker-library/official-images. Following this practices we can create lightweight containers (with a small footprint), easy to use and easy to update for anyone.

  Stackstorm packages for debian platform should be brushed up to eliminate magic around installation, so bringing up st2actions is as easy as `apt-get install st2actions`. Other platforms (rpm-based) no need to tackle right away, it doesn't block delivering of containers. However the brush up methods for debian platform can be useful for creating *proper* packages for other platforms as well.
