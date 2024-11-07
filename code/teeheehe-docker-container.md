```sh
models@PC-STREAM:~$ docker image inspect teeheehee/err_err_ttyl
[
    {
        "Id": "sha256:5ba7efea026e55db62087f2e881d02c459503e1daf3f9f146bc2d5e519f14928",
        "RepoTags": [
            "teeheehee/err_err_ttyl:latest"
        ],
        "RepoDigests": [
            "teeheehee/err_err_ttyl@sha256:50c3f8a00bbc47533504b026698e1e6409b4938109506c4e5a3baaae95116eb7"
        ],
        "Parent": "",
        "Comment": "buildkit.dockerfile.v0",
        "Created": "2024-10-29T18:06:42.435689978-04:00",
        "DockerVersion": "",
        "Author": "",
        "Config": {
            "Hostname": "",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [
                "PATH=/root/.cargo/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "PYTHONUNBUFFERED=1"
            ],
            "Cmd": [
                "bash",
                "run.sh"
            ],
            "ArgsEscaped": true,
            "Image": "",
            "Volumes": null,
            "WorkingDir": "/workdir",
            "Entrypoint": null,
            "OnBuild": null,
            "Labels": {
                "org.opencontainers.image.ref.name": "ubuntu",
                "org.opencontainers.image.version": "22.04"
            }
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 3536559518,
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/335fab7cb0adacfd14b4b51c5e145bb530203e092d6db61d5c54efa16bee63fa/diff:/var/lib/docker/overlay2/108d947de6668418e633d8b4b0cade2eb8afb014fce814258c4bff63df139053/diff:/var/lib/docker/overlay2/6302df00baf6a4d30d5602facc92b0605f3cefc7365ac3f6de4f83f6cd3a4955/diff:/var/lib/docker/overlay2/10f20d7bbf6da8889e54af70133c6c0e8fc096fed2ed53aae8c8d00fd1b667ae/diff:/var/lib/docker/overlay2/c06f25fc50243ef0bfc7bfaaf86bdec6473b1e5624905653761bc97d236774e7/diff:/var/lib/docker/overlay2/b44e2213ac34d973504db744434a4e1e981a6b656a1e674268e5fd0b25c8a0d5/diff:/var/lib/docker/overlay2/d4a5e94538ae680b2ed10c2f1bf3f7907eb85a02f8972a67add03682a8bb2cd3/diff:/var/lib/docker/overlay2/23fb2bbcfc72d57191045d6e6fb6068c544b72324e1f28781a85d6ba64637fd6/diff:/var/lib/docker/overlay2/db3770094b94c8a99ceed44c95d95c159c8221ce7983ad6a3c81c8c102944853/diff:/var/lib/docker/overlay2/7ad0b7ab8c437e6d63033dba5bd2ced80488c4ad11610ebf5d6ab1c7fc92bc9d/diff:/var/lib/docker/overlay2/c2117b629d765996aecd6a053227615ba6e22b4b469dcd7e904d54d6a538bae0/diff:/var/lib/docker/overlay2/e315f7361345b26aedd7a08891e8a943366ccb3aa15c336a37086afcc0307e76/diff:/var/lib/docker/overlay2/abebcaeb0f2bca3397749036c09443398bc17e76647cc8ce93d617b1784c2e10/diff:/var/lib/docker/overlay2/e2db166be90ca1b5f6744b01bbbe4ba106f5358d32b213048d3bffc7f1c50dcf/diff:/var/lib/docker/overlay2/43100155e4f30904ea6cfeb4d23281ec3f5df58d73920c476221a7f37e4e9ead/diff:/var/lib/docker/overlay2/f31883a38bf9ef6e59c789af63bac53c03e4dcaa1d7d6691a04f89126b837662/diff:/var/lib/docker/overlay2/bfd8f0d78b1d492ba36a42d709438686d6d3c0da718c56301163458fb5487839/diff:/var/lib/docker/overlay2/d0de13dc699808c4d327d5c6029435b4eeb8fe5ac215347c51b7d623f215aa6f/diff:/var/lib/docker/overlay2/2cecfc54c2e7abe32cc3cc1c1d40f0624d7ee0aa2da22309bbb90e12b09e00be/diff:/var/lib/docker/overlay2/6ec13abe4188d0d72095c178bd19c4ca63d5d4fa7bca24b12b2ea3bbfddc0d87/diff:/var/lib/docker/overlay2/aac925362b2c070c2a7eac964b41a0c766c6813ebb3561e704e35dd58485f662/diff:/var/lib/docker/overlay2/db4ced19d55e2586e3e5b11d04386bceb528e97aec0782ca928196c128f8cf4d/diff:/var/lib/docker/overlay2/e0fcde18c50812f4030309a95f35e4c61cbf1515ed38caeb0e1b199429ed6993/diff:/var/lib/docker/overlay2/f0f7b633ba30ba5c8f659c4e22f6410dad8b1e75c4db9b58621b92e87e932db4/diff:/var/lib/docker/overlay2/cbd145440e4626d080cdab9eed1920bb3a29d3f72df4f3bda6fee140b138d906/diff:/var/lib/docker/overlay2/3636ce8eb8fc6e32aa5d3577ed0d85864407300aee9120ec1c85c478a7014064/diff:/var/lib/docker/overlay2/f8331e797e724dea9d115ed0aff9cda83ab20dbbc9ed5358be847cc2a92a0f4e/diff:/var/lib/docker/overlay2/e56840b14d4b1743ab698ca21b8b79eab1154cb9416ebab633166e2789e4fe7e/diff",
                "MergedDir": "/var/lib/docker/overlay2/1843a934ceaf2f09bac8b7658036fa2ae1ca5129fb248ad51299189217894767/merged",
                "UpperDir": "/var/lib/docker/overlay2/1843a934ceaf2f09bac8b7658036fa2ae1ca5129fb248ad51299189217894767/diff",
                "WorkDir": "/var/lib/docker/overlay2/1843a934ceaf2f09bac8b7658036fa2ae1ca5129fb248ad51299189217894767/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:2573e0d8158209ed54ab25c87bcdcb00bd3d2539246960a3d592a1c599d70465",
                "sha256:edb4ff6d6f5765e967a6d734344b8154807b9a9f13b224a10812b58a49a4c035",
                "sha256:8e6654a6e13da907d4d9c4922ec12c35c586d7b70360fb3b3ee12521cc0462c5",
                "sha256:da552faaa2ef8421d039507fa389f093ee681a4631b27cc1bf60a590a303c994",
                "sha256:a9ae071fb2ac48bbd3ad6bcd8c2ea3efd073da8ac904eb466374d4bdb542bbeb",
                "sha256:a5c20d936fab37adf9b1eaaf5e749b8a3e3eb4387604093722e1192373cc75c6",
                "sha256:96dc05cb0d59a6b9b09e00d4735d3a0ec008a8b70d5fabf1c625b274cbac0e6f",
                "sha256:05a8f2703050bc35bb9f536d4983f9181a0fe29b04ace74fd4409bcce7e47900",
                "sha256:5159648c51226d66e6a62fc29fb198c11965c6ea11f43e85233f5d12fe9f5512",
                "sha256:c601bffec4dc21457b2e5b0a370fb6657756c0b8f17da1a03dbec471d4e7ba36",
                "sha256:c1a3f47e64efe4090fda7d888b6a9461e9fdca2321dc65e499c12aa55caa426b",
                "sha256:16ed9de4f92fc3428f932c9a52f3a743c669a5e9622b5186d815bb1f3b56611f",
                "sha256:57344c9920e6bf53f1e367171fab94154f2c65d738acc73ce3e77280f3e18a48",
                "sha256:25a83bb18a98b82dfe1722f519dc174abe85252e02b35689165928a061349ba7",
                "sha256:d3d5f6e73ba72818ebc7f7968b13445e48ef4e7612b7a5668c09f53266ef4f2a",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:bc0c2c49ff1593650b097b901c2fbebf6b798e176d79cccffc2c7d85d2a0db82",
                "sha256:18215a36facedebfa1f71550921c68fb6a0c0b46086e8a0b82c8e1344d1c110f",
                "sha256:ac13a9d835febff277883fc5c57f6bc25c3a73ebd28c351b2c7b9e7a22558631",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:61df72e2d1f110402a05792df3a3558bb8f365c53b0639ecc7ba24653a585836",
                "sha256:53477cb8fd64ba27c391ccfeb31c294e83d8568fbd3acade0ff11103f5e30682",
                "sha256:00f879f64afaea597411a41465bf768544221daac63df32c333bc4769890c9e4",
                "sha256:eeb4b1588de768b4b1a163e780a698dccf3eb958ed12bfbbc7dcd928a9018b15",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:9c46fcfc2c6a9c8d691a1fa681a9c2b8524526c885da0dbadaa701edc509bc32",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef",
                "sha256:70f5c0899b366abbe8abc267852516da512695a5c83358084ad3f00170bf9951",
                "sha256:4252145f128d5ac2530ed3fc9c19285941ef9bb94040f860f139c31ac4d692a4"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        }
    }
]
models@PC-STREAM:~$ docker history teeheehee/err_err_ttyl
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
5ba7efea026e   8 days ago    CMD ["bash" "run.sh"]                           0B        buildkit.dockerfile.v0
<missing>      8 days ago    ENTRYPOINT []                                   0B        buildkit.dockerfile.v0
<missing>      8 days ago    RUN /bin/sh -c mkdir -p /data # buildkit        0B        buildkit.dockerfile.v0
<missing>      8 days ago    COPY scripts/ ./scripts/ # buildkit             24.4kB    buildkit.dockerfile.v0
<missing>      8 days ago    WORKDIR /workdir                                0B        buildkit.dockerfile.v0
<missing>      8 days ago    RUN /bin/sh -c cargo build --release # build…   20.8MB    buildkit.dockerfile.v0
<missing>      8 days ago    WORKDIR /workdir/client                         0B        buildkit.dockerfile.v0
<missing>      8 days ago    COPY timerelease.sh ./ # buildkit               1.17kB    buildkit.dockerfile.v0
<missing>      8 days ago    COPY run.sh ./ # buildkit                       1.35kB    buildkit.dockerfile.v0
<missing>      8 days ago    COPY nousflash/agent/ ./agent/ # buildkit       376kB     buildkit.dockerfile.v0
<missing>      9 days ago    COPY client/ ./client/ # buildkit               155MB     buildkit.dockerfile.v0
<missing>      9 days ago    WORKDIR /workdir                                0B        buildkit.dockerfile.v0
<missing>      9 days ago    RUN /bin/sh -c rm -r src # buildkit             0B        buildkit.dockerfile.v0
<missing>      9 days ago    RUN /bin/sh -c cargo build --release # build…   580MB     buildkit.dockerfile.v0
<missing>      9 days ago    RUN /bin/sh -c mkdir src && echo "fn main() …   13B       buildkit.dockerfile.v0
<missing>      9 days ago    WORKDIR /workdir/client                         0B        buildkit.dockerfile.v0
<missing>      9 days ago    COPY client/Cargo.lock client/Cargo.toml cli…   75.4kB    buildkit.dockerfile.v0
<missing>      9 days ago    RUN /bin/sh -c mkdir client # buildkit          0B        buildkit.dockerfile.v0
<missing>      9 days ago    RUN /bin/sh -c pip install -r requirements.t…   275MB     buildkit.dockerfile.v0
<missing>      9 days ago    COPY requirements.txt ./ # buildkit             250B      buildkit.dockerfile.v0
<missing>      11 days ago   ENV PYTHONUNBUFFERED=1                          0B        buildkit.dockerfile.v0
<missing>      11 days ago   WORKDIR /workdir                                0B        buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c apt-get install -y jq # build…   2.79MB    buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c apt-get install -y pkg-config…   14.1MB    buildkit.dockerfile.v0
<missing>      11 days ago   ENV PATH=/root/.cargo/bin:/usr/local/sbin:/u…   0B        buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c curl https://sh.rustup.rs -sS…   1.24GB    buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c apt-get install -y libasound2…   3.78MB    buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c apt-get install -y ./chromium…   646MB     buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c curl -LO https://freeshell.de…   107MB     buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c apt-get install -y curl # bui…   3.56MB    buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c apt-get install -y python3-pi…   348MB     buildkit.dockerfile.v0
<missing>      11 days ago   RUN /bin/sh -c apt-get update # buildkit        56.5MB    buildkit.dockerfile.v0
<missing>      8 weeks ago   /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B
<missing>      8 weeks ago   /bin/sh -c #(nop) ADD file:ebe009f86035c175b…   77.9MB
<missing>      8 weeks ago   /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B
<missing>      8 weeks ago   /bin/sh -c #(nop)  LABEL org.opencontainers.…   0B
<missing>      8 weeks ago   /bin/sh -c #(nop)  ARG LAUNCHPAD_BUILD_ARCH     0B
<missing>      8 weeks ago   /bin/sh -c #(nop)  ARG RELEASE                  0B
models@PC-STREAM:~$
```