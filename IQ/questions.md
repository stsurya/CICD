## scheduling workflow in github actions

'''
name: Scheduled Job

on:
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

jobs:
  run-script:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run script
        run: echo "Scheduled workflow running"
  '''
