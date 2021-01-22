## test-ODP

This repo aims at illustrating the production of ODP format datasets in _GitHub_ through automatically triggered workflwow/actions.

**Source material (dataset + code): https://github.com/ceperezegma/ODP_datasets_distributions.**

_objective:_ Automatically run the production script at 8:00am every morning.

_constraints:_
1. input datasets are stored in a _Google_ spreadsheet,
2. input datasets are fetched through _Google_ API through service authentication,
3. operations are run from a `Jupyter` notebook (rather than using a script or a library): **[`ODP_datasets_formats_update.ipynb`](https://github.com/ceperezegma/ODP_datasets_distributions/blob/main/ODP_datasets_formats_update.ipynb)** ([preview](https://nbviewer.jupyter.org/github/ceperezegma/ODP_datasets_distributions/blob/main/ODP_datasets_formats_update.ipynb))

_resolution:_
* the **updated notebook [`ODP_datasets_formats_update.ipynb`](./ODP_datasets_formats_update.ipynb)** ([preview](https://nbviewer.jupyter.org/github/gjacopo/test-ODP/blob/main/ODP_datasets_formats_update.ipynb)) enables to circumvent constraint \#2 by accessing and fetching the data from the _Google_ spreadsheet without the need to authenticate:
  * the **function `gsheet_into_df`** is defined in place of `set_google_drive_access_scope`, `access_google_sheet` and `last_processed_date` to avoid using _Google_ API and service authentication:
  ```python
   def gsheet_into_df(urlbase, sheet_key, worksheet_id, file, first_column, last_column):

      url_domain = '%s/ccc?key=%s&gid=%s&output=csv' % (urlbase, sheet_key, worksheet_id)

      try:
          date_parser = lambda x: datetime.datetime.strptime(x, "%d/%m/%Y")
          last_proc_dates = pd.read_csv(file, header = 0, parse_dates = [0], date_parser = date_parser)
      except:
          print('File %s not found -- All data will be loaded' % file)
          next_row_to_read = 0
      else:
          next_row_to_read = last_proc_dates.shape[0] + 2

      try:
          url = '%s&range=A:A' % url_domain
          r = requests.get(url)
      except:
          raise IOError('URL %s not reached' % url)
        
      try:
          date_parser = lambda x: datetime.datetime.strptime(x, "%m/%d/%Y %H:%M:%S")
          dates = pd.read_csv(BytesIO(r.content), 
                              parse_dates = ['1'], date_parser = date_parser) 
      except:
          raise IOError('Data not loaded')
      else:
          row_total = len(dates)

      if row_total < next_row_to_read:  
          return None # [0]     
    
      try:
          cell_range = '%s%s:%s%s' % (first_column, next_row_to_read, last_column, row_total)
          url = '%s&range=%s' % (url_domain, cell_range)
          r = requests.get(url)
      except:
          raise IOError('URL %s not reached' % url)
      else:
          df = pd.read_csv(BytesIO(r.content), header = None) 
        
      return df
  ```
  * packages `gspread` and `oauth2client` are discarded since no authentication is required; instead `requests` objects are used,
  * package `tqdm` is also discarded for compatibility with the workflow.
  * references to local data/variables are discarded (*e.g.*, local path, local credentials data, *etc...*),
* the **action configuration file [`automated.yml`](.github/workflows/automated.yml)** is used to automatically run the production of ODP:
  * the repo is checked out (see [actions marketplace](https://github.com/actions/checkout)):
  ```yaml
        uses: actions/checkout@v2 
      with:
        persist-credentials: false
        fetch-depth: 0
  ```
  * `Python` (version 3.8) is set up (see [actions marketplace](https://github.com/actions/setup-python)):
  ```yaml
        uses: actions/setup-python@v2 
      with:
        python-version: ${{ matrix.python-version }}
  ```
  * the environment packages are installed:
  ```yaml
      run: |
        python -m pip install --upgrade pip
        if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
  ```
  * `nbconvert` is installed:
  ```yaml
      run: |
        python -m pip install --upgrade pip
        pip install nbconvert
  ```
  * the notebook `ODP_datasets_formats_update.ipynb` is converted to a `Python` script:
  ```yaml
      run: |
        jupyter nbconvert --to script ODP_datasets_formats_update.ipynb
  ```
  * the generated `Python` script `ODP_datasets_formats_update.py` is run:
  ```yaml
      run: |
        python ODP_datasets_formats_update.py
  ```
  * output datasets are committed back to the repo:
  ```yaml
      uses: EndBug/add-and-commit@v4
      with:
        message: "Automated data fetching"
        add: 'datasets_formats_processed.csv' 
        force: true
  ```
  * the jobs above are scheduled to run every day at 7:00am UTC (8:00):
  ```yaml
   - cron: "0 7 * * *"
  ``` 
      or to be automatically triggered upon a change of the source notebook:
  ```yaml
   push:
     paths:
       - '**.ipynb'
  ```
