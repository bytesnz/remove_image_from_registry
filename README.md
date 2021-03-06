# Remove a image from a remote docker registry

This bash script makes it possible to delete a image from a docker registry v2
Don't be afraid to delete images which contains layers used by other images. The docker registry backend will automatically detect that and keep those layers.

## Quick start

```sh
 $ docker run --rm -ti vidarl/remove_image_from_registry -u username -p myregistry:5000/myrepo/myimage:latest
```

## This script requires:
* Docker registry (v2)
* Registry must use token based authentication
   Check out https://github.com/cesanta/docker_auth for a nice authentication server.
* Deletion must be explicitly enabled on the registry server
  * Use environment variable REGISTRY_STORAGE_DELETE_ENABLED=true
  * Or storage.delete.enabled=true in [config.yml](https://docs.docker.com/registry/configuration/#delete).

# Usage

```sh
 $ ./remove_image_from_registry.sh [OPTIONS] [IMAGE]

IMAGE
 Image name has the format registryhost:port/repository/imagename:version
 For instance : mydockerregistry:5000/myrepo/zoombie:latest
 Note that the version tag ("latest" in this example) is mandatory.
 Please note that this script will delete the image from the repository, not only the tag; if the
 image you are deleting have multiple tags ( for instance "1.0" and "latest" ), both tags will be
 removed from the registry.
 Docker registry does not support deleting only a tag ATM, ref https://github.com/docker/distribution/issues/2317
 The option "--tag-only" tries to circumvent this by restoring other tags which also disappear from
 the registry during the delete. However, this is not entirely safe:
  - Do not delete multiple images concurrently.
  - Do not run registry garbage collector while deleting images (this you should never do anyway....).
  - Do not create new local tags for the image during the delete operation.
 
OPTIONS
 -h, --help
        Print help
 --insecure
        Connect to a registry which has a self-signed SSL certificate
 -p
        Prompt for password
 -u <username>
        Use the given username when authenticating with the registry
 --raw <url> <http-method> [http-header]
         Send custom request to the registry. When using this argument, do not use the  [IMAGE] argument too.
         Example:
         ./remove_image_from_registry.sh \
              -u admin \
              --insecure \
              --raw \
              mydockerregistry:5000/v2/imagename/manifests/latest \
              GET \
              "Accept: application/vnd.docker.distribution.manifest.v2+json"
 --tag-only
        After deleting the image, try to recover all other tags which also pointed to the image


Password may also be set using the environment variable REGISTRY_PASSWORD
 $ export REGISTRY_PASSWORD=sesame
```

## Example
```sh
# Store password in environment variable
export REGISTRY_PASSWORD=foobar
# Delete the image mydockerregistry:5000/myrepo/zoombie:latest
./remove_image_from_registry.sh -u foo --insecure mydockerregistry:5000/myrepo/zoombie:latest
```

Note that this script will not delete the actual blobs from the registry, only the manifests. Once you have deleted the manifests you have to manually run the garbage collector in order to delete the blobs. You do so by entering your registry container:

```sh
docker exec -ti registry_registry_1 /bin/sh
bin/registry garbage-collect /etc/docker/registry/config.yml
```

See [the documentation](https://docs.docker.com/registry/garbage-collection/) for more information about the garabe collector.



# Permissions in registry
You must have all (`["*"]`) privileges in order to delete an image, `['push']` is *not* sufficient.
However, using docker_auth you can give access per repository and image 

```yaml
### auth_config.yml
users
  "foobar":
    password: "...."
  "jane":
    password: "...."
acl:
  - match: {account: "foobar", name: "myrepo/zoombie"}
    actions: ["*"]
    comment: "foobar can do anything with image myrepo/zoombie"
  - match: {account: "jane", name: "myrepo/*"}
    actions: ["*"]
    comment: "jane can do anything with any image in the repo named myrepo"
```

# Relevant registry bugs
Be aware of a [cache bug](https://github.com/docker/distribution/issues/2094#issuecomment-326454550) in the garbage collector.
As a consequence of this bug, you'll need to restart your registry after running the garbage collector (or clear redis cache if you are using that).

Also note that when deleting a image, [the repository will still remain in the catalogue](https://github.com/docker/distribution/issues/2314).

If you use docker client to push and overwrite an existing tag in the registry, the garbage collector will *not*
remove the blobs beloning to the overwritten image. [This bug comment](https://github.com/docker/distribution/issues/2212#issuecomment-292021283) explains why.
