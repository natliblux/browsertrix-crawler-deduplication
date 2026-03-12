# Introduction

Automatic quality assurance is already provided by default by Browsertrix crawler. In this fork, we introduce a few refinements to this QA workflow:

  1. The ability to set a maximum page count for the QA,
  2. The ability to choose between 3 different QA algorithms.
  
These algorithms are: *linear* (first come first serve), *regex* (define a regular expression to perform QA only on URLs that match it), and *random* (define a probability for each page to be QA'd).

These features are intended to be used together with the Browsertrix UI (formerly *Browsertrix Cloud*), by adjusting the additional crawler parameters in the local `config.yaml` in the Kubernetes deployment. A small guide is provided at the end of this readme.
  
# New CLI arguments

We provide the following new CLI arguments for the crawler:

  - `qaMaxUrls`: the maximum number of pages to perform QA on,
  - `qaPolicy`: can be one of `linear`, `regex` or `random`,
  - `qaRegex`: if `qaPolicy` is `regex`, then you can define your regular expression here,
  - `qaProbability`: if `qaPolicy` is `random`, you can define your per-page QA probability here. This is a floating-point number between 0 and 1.
  
### QA Policy: `linear`

In this QA mode, the first `qaMaxUrls` pages in the `pages.jsonl` file(s) will be scanned. Example:

    --qaPolicy "linear" --qaMaxUrls 50"
    
### QA Policy: `regex`

In this QA mode, only the pages that match the regular expression in `qaRegex` will be scanned. Example:

    --qaPolicy "regex" --qaRegex='^https:\/\/en\.wikipedia\.org\/wiki\/R.*$' --qaMaxUrls 50"
    
This will match all english Wikipedia articles that start with `R`.

### QA policy: `random`

In this QA mode, pages will be scanned with a probability equal to `qaProbability`. This is a floating-point number between `0` and `1`. Example:

    --qaPolicy "random" --qaProbability 0.3 --qaMaxUrls 50"
    
This policy allows to get a good overall impression of a harvest by scanning a random sample of it.
    
### Maximum number of pages to scan

In every case mentioned above, the number of pages that will be queued for scanning will be at most `qaMaxUrls`.

# Usage with the Browsertrix UI

You can easily use these features with Browsertrix UI (*Cloud*) in the following manner:

  1. Compile the crawler as usual,
  2. Tag it as you wish and push it to local registry,
  3. Adapt the `crawler_channels` in your deployment's `local-config.yaml` file so that it points to the crawler in your registry,
  4. Add your QA policy and other parameters to the `crawler_extra_args` variable,
  5. (optional) Clear your Kubernetes/microk8s cache with `microk8s ctr images rm localhost:32000/<your-crawler-image>`
  6. Reload your deployment.

Now whenever you will start a new QA workflow from the Browsertrix Cloud interface, the crawler instance that will be spawned will already be running the new QA workflow with your specified parameters.



