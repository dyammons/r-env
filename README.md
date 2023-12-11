# r-env | A singularity container for R analysis 


### Create initial singularity image that has base R packages installed
To build the inital container vist [Syslabs](https://cloud.sylabs.io/) and use the remote build option. The def file used to build the environment is provided below, so copy the contents into the builder and start the build.

```sh
BootStrap: docker
From: ubuntu:20.04

%labels
  Maintainer Dylan Ammons (adapted from Jeremy Nicklas)
  R_Version 4.3.2

%apprun R
  exec R "${@}"

%apprun Rscript
  exec Rscript "${@}"

%runscript
  exec R "${@}"

%post
  # Software versions
  export R_VERSION=4.3.2
  echo "export R_VERSION=${R_VERSION}" >> $SINGULARITY_ENVIRONMENT

  # Get dependencies
  apt-get update
  apt-get install -y --no-install-recommends \
    locales

  # Configure default locale
  echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
  locale-gen en_US.utf8
  /usr/sbin/update-locale LANG=en_US.UTF-8
  export LC_ALL=en_US.UTF-8
  export LANG=en_US.UTF-8

  # Install R
  apt-get update
  apt-get install -y --no-install-recommends \
    software-properties-common \
    dirmngr \
    wget
  wget -qO- https://cloud.r-project.org/bin/linux/ubuntu/marutter_pubkey.asc | \
    tee -a /etc/apt/trusted.gpg.d/cran_ubuntu_key.asc
  add-apt-repository \
    "deb https://cloud.r-project.org/bin/linux/ubuntu $(lsb_release -cs)-cran40/"
  apt-get install -y --no-install-recommends \
    libharfbuzz-dev \
    libfribidi-dev \
    r-base-dev=${R_VERSION}* \
    r-cran-lattice \
    r-cran-mgcv \
    r-cran-nlme \
    r-cran-survival \
    r-cran-mass \
    r-cran-class \
    r-cran-nnet \
    r-cran-matrix \
    r-recommended=${R_VERSION}* \
    r-base=${R_VERSION}* \
    r-base-core=${R_VERSION}* \
    libcurl4-openssl-dev \
    libssl-dev \
    libxml2-dev \
    libcairo2-dev \
    libxt-dev \
    libopenblas-dev \
    gcc \
    g++ \
    libfreetype6-dev \
    libglib2.0-dev \
    libpng-dev \
    libtiff5-dev \
    libjpeg-dev \
    libhdf5-dev \
    cmake

  # Add a default CRAN mirror
  echo "options(repos = c(CRAN = 'https://cran.rstudio.com/'), download.file.method = 'libcurl')" >> /usr/lib/R/etc/Rprofile.site

  # Add a directory for host R libraries
  mkdir -p /library
  echo "R_LIBS_SITE=/library:\${R_LIBS_SITE}" >> /usr/lib/R/etc/Renviron.site

  # Clean up
  rm -rf /var/lib/apt/lists/*

%test

R --quiet -e "stopifnot(getRversion() == '${R_VERSION}')"
```

### Get sif and install required R packages

To retreive the new base container, navigate to the repository and find the `singularity pull` command that indicates how to retrieve the container.
```sh
#should look like this
singularity pull library://dyammons/r-env/r4.3.2-seuratv5
```

Once the `singularity pull` command has been located, you can create a directory to work from where you can install the required R packages. For example consider the following directory structure:
```sh
mkdir -p /projects/$USER/software/singularity/
cd /projects/$USER/software/singularity/
```

While in the `sandbox` directory you will use your repository's specific `singularity pull` command.
```sh
#pull down your repository, so modify the below command as needed
singularity pull library://dyammons/r-env/r4.3.2-seuratv5
```

Once the container is there you will see a `.sif` file. In this example it is called `r4.3.2-seuratv5_latest.sif`.
```sh
ls
#r4.3.2-seuratv5_latest.sif
```

Because the `.sif` file is an immutable object we must convert the object to a format that enables the installation of additional software.  
To do this we will convert the file into a writable `sandbox` which is a version of the container that can be modified by adding/deleting files from the directory structure.
```sh
#first argument is the name of the sandbox, second argument is the name of the .sif to convert
singularity build --sandbox r4.3.2-seuratv5 r4.3.2-seuratv5_latest.sif
```

Now the sandbox should be built and you can see a new directory called `r4.3.2-seuratv5`.
```sh
ls
#r4.3.2-seuratv5 r4.3.2-seuratv5_latest.sif
```

We can now launch the container through the sandbox and see that base R is properly installed.
```sh
singularity run r4.3.2-seuratv5
```

This will launch an R session and we will use a R session to install packages. However, if you try to complete install packages now using `install.packages()` you will see it cannot write to the container to install the new packages. To fix this we will need to enter the container through use of writable shell session. So, let's exit the R session we are currently in.
```r
#quit R
q()

#do not save session
n
```

```sh
#launch a writable session to install packages
singularity shell --writable r4.3.2-seuratv5
```

Once inside this shell we can install packages directly through the terminal with commands such as the following:
```sh
#example command; change packages to what is needed
R --quiet --slave -e "install.packages(c('Seurat','clustree'))"
```

<details>
  <summary>Click for exact steps used in r4.3.2-seuratv5 build</summary>
<p>

<br>

```r
#install completed on 12.10.2023 - DA

#attempt to install all at once -- several failed, so reorder in future
R --quiet --slave -e "install.packages(c('Rcpp', 'ggforce', 'ggrepel', 'graphlayouts','sitmo','dqrng','uwot','devtools','lme4','tidyverse','clustree','stringr','remotes','patchwork','scales','cowplot','ggrepel','colorspace','BiocManager','pheatmap','RColorBrewer','viridis','reshape','lemon','msigdbr','ggpubr','ape','UpSetR','Seurat'))"
R --quiet --slave -e "install.packages(c('RSpectra', 'ggforce', 'ggraph','Seurat','clustree'))"
R --quiet --slave -e "install.packages(c('pbkrtest', 'car', 'rstatix','R.utils','ape','tidytree','circlize','RColorBrewer'))"

#step through one by one to catch dependency errors 
R --quiet --slave -e "BiocManager::install(c('limma','DESeq2'))"
R --quiet --slave -e "BiocManager::install(c('beachmat','BiocSingular','SingleR','celldex','treeio','ggtree','enrichplot','clusterProfiler','slingshot','scRNAseq','scuttle','ComplexHeatmap'))"
R --quiet --slave -e "remotes::install_github('chris-mcginnis-ucsf/DoubletFinder')"
R --quiet --slave -e "remotes::install_github('mojaveazure/seurat-disk')"
R --quiet --slave -e "devtools::install_github('davidsjoberg/ggsankey')"
R --quiet --slave -e "devtools::install_github('rpolicastro/scProportionTest')"
R --quiet --slave -e "devtools::install_github('immunogenomics/presto')"
R --quiet --slave -e "devtools::install_github('jinworks/CellChat')"
R --quiet --slave -e "devtools::install_github('arc85/singleseqgset')"
R --quiet --slave -e "BiocManager::install(c('beachmat','BiocSingular','SingleR','scuttle'))"
R --quiet --slave -e "BiocManager::install('BiocSingular')"
R --quiet --slave -e "BiocManager::install('SingleR')"
R --quiet --slave -e "install.packages('harmony')"
```

</p>
</details>

The `R` packages are now installed, so we can now finalize the `.sif` and pack it up to share internally or externally. 

```sh
#pack up the container - this could take some time
singularity build r4.3.2-seurat5.sif r4_3_2s5/
```

If wanted to make the container accessible to anyone with internet, continue on, otherwise you can stop here and use your container however you desire. The `.sif` file contains all the software information, so if you are done adding packages to the container you can delete the sandbox.  

The packages `.sif` file can be sent up to the Syslabs repository for easy sharing. To do this, you will need to create a Syslabs access token. See the Syslabs [documention](https://docs.sylabs.io/guides/3.3/user-guide/cloud_library.html#:~:text=Creating%20a%20Access%20token,-Access%20tokens%20for&text=the%20following%20steps%3A-,Go%20to%3A%20https%3A%2F%2Fcloud.sylabs.io%2Flibrary,%E2%80%9CCreate%20a%20new%20token%E2%80%9D) for details.  

In addition to the access key we will need to create a key pair which is a unique value that will allow us to sign the container and forever link the container to you.
```sh
#check if you already have an keys
singularity keys list

#create a new key pair
singularity keys newpair
```

Now you can check that the key exists.
```sh
#check for keys
singularity keys list

#example output
# 0) U: John Doe (my key) <johndoe@sylabs.io>
# C: 2018-08-21 20:14:39 +0200 CEST
# F: D87FE3AF5C1F063FCBCC9B02F812842B5EEE5934
# L: 4096
````

### finish below - got tired haha

```sh
singularity keys push A02599B62A9EF391FDB42E7A404C99A945E84E08
singularity sign r4.3.2-seurat5.sif
singularity push r4.3.2-seurat5.sif library://dyammons/r-env/r4.3.2-seurat5:v2
```


