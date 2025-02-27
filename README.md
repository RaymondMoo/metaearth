## MetaEarth: Download any remote sensing data from any provider using a single config.

<img width="1361" alt="MetaEarth Explainer Diagram - download any data from any provider" src="https://user-images.githubusercontent.com/1455579/180137540-80b749d0-ab3d-469d-8122-f6b1a0df008f.png">


---

**🔥 Warning 🔥** This is a very early alpha version of MetaEarth: things will change quickly, with little/no warning. The current MetaEarth explainer image above is aspirational: we're actively working on adding more data providers.

---

## Quick Start

Install MetaEarth as a library and download about 18MB of [Copernicus DEM](https://planetarycomputer.microsoft.com/dataset/cop-dem-glo-90) data from Microsoft Planetary Computer -- this small example should Just Work™ without any additional authentication.

```bash
# OPTIONAL: set up a conda environment (use at least python 3.7)
conda create -n metaearth python=3.8 geopandas
conda activate metaearth

git clone git@github.com:bair-climate-initiative/metaearth.git
cd metaearth
pip install -e . 

# Take a look at the download using a dry run (you could also set dry_run in the config file):
python metaearth/cli.py --config config/demo.yaml system.dry_run=True

# If everything looks good, remove the dry_run and download Copernicus DEM data from Microsoft Planetary Computer
python metaearth/cli.py --config config/demo.yaml

# see the extracted data in the output directory
ls data/demo-extraction-dem-glo-90/cop-dem-glo-90/
```

**Quick Explanation:** The config we're providing, [config/demo.yaml](config/demo.yaml), contains a fully annotated example: take a look at it to get a sense of config options and how to control MetaEarth. While playing with MetaEarth, set the dryrun config option in order to display a summary of the assets without downloading anything, e.g. `system.dry_run=True`. Note that to download more/different data from Microsoft Planetary Computer, you'll want to authenticate with them (see the instructions under [Provider Configurations](#provider-configurations)).


## Documentation
**🔥 Warning 🔥** The documentation is intentionally sparse at the moment: MetaEarth is under rapid development and writing/re-updating the documentation during this period would be more effort than benefit. 

See the *Quick Start* instructions above and then consult [config/demo.yaml](config/demo.yaml) for annotated configuration (we'll keep this annotated config updated).


### MetaEarth Configuration
The following describes some common goals for configuring MetaEarth, such as specifying a data collection, geographical region, and timerange to extract data from, or specifying a provider to download data from. Consult [config/demo.yaml](config/demo.yaml) for an annotated configuration. The configuration schemas are defined in [metaearth/config.py](metaearth/config.py): take a look at `ConfigSchema`. 

**Specifying the data to be downloaded** takes place through the `collections` config option. For instance, to download [Copernicus DEM](https://planetarycomputer.microsoft.com/dataset/cop-dem-glo-90), which has the collection id `cop-dem-glo-90` (see below on how to find this), the config is like this:
```yaml
collections: 
  cop-dem-glo-90:
    # specify which assets to download. 
    # use a single entry with "all" to download all assets.
    assets:
      - data
    # the data will output to this directory.
    outdir: data/demo-extraction-dem-glo-90
    # Single date+time, or a range ('/' separator), 
    # formatted to RFC 3339, section 5.6. 
    # Use double dots .. for open date ranges.
      datetime: "2021-04-01/2021-04-23"
    # area-of-interest file location
    # this demo contains a small section in Yosemite, 
    # view by pasing demo.json into http://geojson.io
    aoi_file: config/aoi/demo.json
    provider: 
        # MPC is the identifier for Microsoft Planetary Computer
        # See "provider key" under "Provider Configurations"
        name: MPC
```


**Finding the collection id**: This depends on the individual provider (see [Provider Configurations](#provider-configurations) below).

**Finding the assets**:
This depends on the individual provider (see [Provider Configurations](in the future, see #provider-configurations) below), but the following seems to be a pretty solid method:

1. Create a config with your desired collection id, set the `assets` option to `["all"]` like this (and setting `max_items` to 1 to speed things up):
```yaml
collections: 
  landsat-8-c2-l2:
    assets:
      - all
    max_items: 1
...
```
2. Run a dry run to see what assets will be downloaded:
```bash
python metaearth/cli.py --config path/to/your/config.yaml system.dry_run=True
```
which will print out a list of assets that will be downloaded and their descriptions, e.g.:
```
18:24:51 INFO Asset types:
key=ANG; desc="Collection 2 Level-1 Angle Coefficients File (ANG)"
key=SR_B1; desc="Collection 2 Level-2 Coastal/Aerosol Band (B1) Surface Reflectance"
key=SR_B2; desc="Collection 2 Level-2 Blue Band (B2) Surface Reflectance"
key=SR_B3; desc="Collection 2 Level-2 Green Band (B3) Surface Reflectance"
key=SR_B4; desc="Collection 2 Level-2 Red Band (B4) Surface Reflectance"
key=SR_B5; desc="Collection 2 Level-2 Near Infrared Band 0.8 (B5) Surface Reflectance"
key=SR_B6; desc="Collection 2 Level-2 Short-wave Infrared Band 1.6 (B6) Surface Reflectance"
...
```
3. Let's say we want the RGB channels (see the descriptions), so we then update our config to download only the assets we want, and remove the `max_items` option:
```yaml
collections: 
  landsat-8-c2-l2:
    assets:
      - SR_B2
      - SR_B3
      - SR_B4
    max_items: 1
...
```

**Selecting a Region and Timerange**
Specify region:

1. Use (something like) https://geojson.io to specify the region you care about in geojson format
1. Save to file, e.g. `my_region.json`
1. Set that file to the `aoi_file` key under the collection you want to extract or to `default_collection` if you want to extract it for multiple collections.

Specify a timerange by using single date+time, or a range ('/' separator), formatted to [RFC 3339, section 5.6](https://datatracker.ietf.org/doc/html/rfc3339#section-5.6). See [config/example.yaml](config/example.yaml) and it should be pretty clear. Use double dots .. for open date ranges.


**Output Directory and Data Format**:
The saved data will be placed in the directory format `{outdir}/{collection_id}/{item_id}/{asset_id}.{asset_appendix}`. 


**Defaults when downloading multiple collections**
You can specify a `default_collection` in your config, which will be inherited by all collections that don't specify a specific key, e.g.
```yaml
# fallback for each collection
# where each of these entries can be overridden 
# in each collection config under "collections"
default_collection:
  # will output to ${output}/collection_name/ by default, can override as an entry in the collection config
  outdir: data/demo-extraction
  # default datetime range for each collection, 
  # can override as an entry in the collection config
  # Single date+time, or a range ('/' separator), 
  # formatted to RFC 3339, section 5.6. 
  # Use double dots .. for open date ranges.
  datetime: 2021-04-01/2021-04-23
  # default aoi for each collection (use geojson format - see geojson.io)
  # can override as an entry in the collection config
  # this demo contains a small section in Yosemite
  aoi_file: config/aoi/demo.json
  # Max number of items 
  # (not assets, e.g. each item could have 3 images)
  # to download. -1 for unlimited (or limit set)
  # by the provider
  max_items: -1
  # default provider for each collection, can override as an entry in the collection config
  provider: 
    # MPC is the identifier for Microsoft Planetary Computer
    name: MPC
```

**Dry run and DEBUG are your friend. You have lots of friends.** When dialing in your configuration, keep the `system.dry_run=True` option on your call to `metaearth/cli.py` (or set it in your config). Also, set the `system.log_level=DEBUG` option to see more verbose output.

### Programmatic API Usage
Programmatic MetaEarth API usage is still under development, but very much a part of our roadmap. For now, you can roughly do the following (let us know if you're interested in API support and how you'd like to use MetaEarth in this context):

```python
from omegaconf import OmegaConf

from metaearth.api import extract_assets
from metaearth.config import ConfigSchema

dict_cfg = {
  "collections": {
    "cop-dem-glo-90": {
      "outdir": "data",
      "assets": ["all"],
      "aoi_file": "config/aoi/demo.json",
      "datetime": "2021-04-01/2021-04-23",
      "provider": {
        "name": "MPC"
      }
    }
  }
}
in_cfg = OmegaConf.create(dict_cfg)
cfg_schema = OmegaConf.structured(ConfigSchema)
cfg = OmegaConf.merge(cfg_schema, in_cfg)
successfully_extracted_assets, failed_assets = extract_assets(cfg)
print(f"Successfully extracted {len(successfully_extracted_assets)} assets. {len(failed_assets)} failed.")
```


## Provider Configurations
---

**🔥 Warning 🔥** This is a very early alpha version of MetaEarth: there's only a few providers and meta-providers supported at the moment. Let us know if you need other providers and we can prioritize adding them.

---

Each provider needs its own authentication and setup. For each provider you'd like to use, follow the set-up instructions below.


### Microsoft Planetary Computer (provider key: MPC)

Make sure to run the following and enter your api key (this helps increase the amount of data you can download from MPC).
```
planetarycomputer configure 
```

**Finding the collection id**: 
1. Go to the [MPC Data Catalog](https://planetarycomputer.microsoft.com/catalog)
1. Find/click-on the desired collection
1. In the Collection Overview Page, click on the "Example Notebook" tab
1. The example notebook will contain an example of accessing the collection using the collection id.


### NASA EarthData (provider key: EARTHDATA)
NASA EarthData provides access to a diverse range of providers (around 60!), where each provider has different data sources.

**Access**
1. For NASA EarthData, you need to create an account at: https://urs.earthdata.nasa.gov/
1. Add a `~/.netrc` file (if it doesn't exist) and then append the following contents:
```
machine urs.earthdata.nasa.gov
    login <username>
    password <password>
```

EarthData is a provider of providers, so you must include a `provider_id` in your `kwargs` argument to the provider, like the following example that accesses ASO data from NSIDC from EarthData ([config/nsidc.yaml](config/nsidc.yaml)):
```
collections: 
  ASO_50M_SD:
    assets:
      - data
  provider: 
    name: EARTHDATA
    kwargs:
      provider_id: NSIDC_ECS
```

**Finding the Provider ID**: Consult [earthdata_providers.py](metaearth/provider/earthdata_providers.py) for a list of providers and their provider ids.

**Finding the collection id**: TODO (this depends on the provider and we need to figure out a general approach)

## Contributing and Development
The general flow for development looks like this:

0. Read the Getting Started Guide - make sure you can sucessfully download some data, and make sure to install this repository in editable mode `pip install -e .`

1. Create a new branch for your feature.

2. Edit the code.

3. Run linters and tests (see subsections below)

4. Commit your changes, push to the branch, and open a pull request. 

5. ???

6. Profit $$$


### Linting
Following [TorchGeo](https://torchgeo.readthedocs.io/en/stable/user/contributing.html#linters) (and literally copying their docs), we use the following linters:

* [black](https://black.readthedocs.io/) for code formatting
* [isort](https://pycqa.github.io/isort/) for import ordering
* [pyupgrade](https://github.com/asottile/pyupgrade) for code formatting

* [flake8](https://flake8.pycqa.org/) for code formatting
* [pydocstyle](https://www.pydocstyle.org/) for docstrings
* [mypy](https://mypy.readthedocs.io/) for static type analysis

Use [git pre-commit hooks](https://pre-commit.com/) to automatically run these checks before each commit. pre-commit is a tool that automatically runs linters locally, so that you don't have to remember to run them manually and then have your code flagged by CI. You can setup pre-commit with:

```
pip install pre-commit
pre-commit install
pre-commit run --all-files
```


## Useful links

* Use https://geojson.io to extract a region-of-interest


### OmegaConf References for Config 
* [Passing in configs](https://omegaconf.readthedocs.io/en/2.2_branch/usage.html?highlight=command%20line#from-command-line-arguments)



## Related Projects

* [Sat-Extractor](https://github.com/FrontierDevelopmentLab/sat-extractor). Sat-Extractor has a similar goal as MetaEarth, though at the moment it has been designed to run on Google Compute Engine, and as of the start of MetaEarth, Sat-Extractor can only be used with Sen2 and LandSats out-of-the-box. By starting MetaEarth with Microsoft's Planetary Computer, MetaEarth immediately has access to their full data catalog: https://planetarycomputer.microsoft.com/catalog (which subsumes the data accessible by Sat-Extractor plus ~100 other sources). Still, Sat-Extractor is an awesome and highly-configurable project: please use and support it if Sat-Extractor aligns with your goals =).

