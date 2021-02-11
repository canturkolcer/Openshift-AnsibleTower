1. Find ocp registry url
    ```
    # oc get route -n openshift-image-registry
    NAME            HOST/PORT                                                                PATH   SERVICES         PORT    TERMINATION   WILDCARD
    default-route   default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>          image-registry   <all>   reencrypt     None
    ```

2. Get certificate for default registry.

    ```
    # openssl s_client -connect default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>:443 -showcerts 2>/dev/null </dev/null | awk '/^.*'"default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>"'/,/-----END CERTIFICATE-----/{next;}/-----BEGIN/,/-----END CERTIFICATE-----/{print}' > /tmp/myca.cert
    # cat /tmp/myca.cert
    .......
    ```

3. Copy image certificate to trust store
    ```
    # cd /etc/pki/ca-trust/source/anchors/
    # ls -altr
    total 12
    drwxr-xr-x. 4 root root 4096 Apr  1 07:58 ..
    lrwxrwxrwx. 1 root root   39 May  6 08:47 RHN-ORG-TRUSTED-SSL-CERT -> /usr/share/rhn/RHN-ORG-TRUSTED-SSL-CERT
    drwxr-xr-x. 2 root root 4096 May 13 08:29 .
    -rw-r--r--. 1 root root 2187 Jun 10 18:12 domain.crt
    # cp /tmp/myca.cert imgRegistry.cert
    ```

4. Update certificates
    ```
    # update-ca-trust
    ```

5. Be Sure your user has access/role or use kubeadmin. Need to get token for user
    ```
    # oc whoami -t
    ```

6. Docker login to push image
    ```
    [root@fre2ins011ccpr1 anchors]# docker login default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>
    Username (serviceaccount): kubeadmin
    Password: <UseTokenAbove>
    Login Succeeded
    ```

7. Find image and tag if needed
    ```
    # docker images
    REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
    ansible-tower             latest               3eec7b129747        5 hours ago         1.55 GB

    # docker tag 3eec7b129747  default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>/tower/tower:3.7.1

    # docker images
    REPOSITORY                                                                                     TAG                 IMAGE ID            CREATED             SIZE
    default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>/tower/tower             3.7.1               3eec7b129747        5 hours ago         1.55 GB

    ```

8. Push image to registry
    ```
    # docker push default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>/tower/tower:3.7.1
    The push refers to a repository [default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>/tower/tower]
    4ffc7221a78e: Pushed
    d39dfe25f585: Pushed
    ab3378e759ad: Pushed
    d62af24c3ddf: Pushed
    f15a9d9f7ab3: Pushed
    cacea99e9a8c: Pushed
    3.7.1: digest: sha256:feb274205e66e2ba816f597c1ab3acd3347d97180bb43c32d5084a7f771ddd3f size: 1583
    ```
9. Get image list
    ```
    # oc get is
    NAME    IMAGE REPOSITORY                                                                     TAGS    UPDATED
    tower   default-route-openshift-image-registry.apps.<OCPDomainPrefix>.<BaseDomain>/tower/tower   3.7.1   About a minute ago
    ```
