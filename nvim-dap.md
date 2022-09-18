# nvim-dap

Documentation on using [nvim-dap](https://github.com/mfussenegger/nvim-dap)

## Guides

Guide(s) followed:
- [Neovim for Beginners — Debugging using DAP](https://alpha2phi.medium.com/neovim-for-beginners-debugging-using-dap-44626a767f57)
- [Neovim DAP Enhanced](https://alpha2phi.medium.com/neovim-dap-enhanced-ebc730ff498b)
- [Neovim for Beginners — Python Remote Debugging](https://alpha2phi.medium.com/neovim-for-beginners-python-remote-debugging-7dac13e2a469)
- [Local and Remote Debugging with Docker](https://github.com/mfussenegger/nvim-dap/wiki/Local-and-Remote-Debugging-with-Docker)

## Configurations

Configuration files perused:
- For [lunarvim](https://www.lunarvim.org): [abzcoding/lvim](https://github.com/abzcoding/lvim/blob/58e6014259937ee99579e9077bb63fdc8a3f1caa/lua/user/dap.lua#L212)

## Overview

This is meant to work with Open Library development, which already forwards the relevant port (here 3000) from local host to Docker, and includes a `.vscode/launch.json` that will work for Python and the local development environment. Links to the Open Library examples are permanent links and the files may have changes not shown in the permanent link.

The guides proved a bit of a distraction when using [Lunarvim](https://www.lunarvim.org). Better to simply install each thing and configure it per the documentation, perhaps. At a high level this includes:
- enabling DAP in Lunarvim
- installing:
  - [nvim-dap](https://github.com/mfussenegger/nvim-dap)
  - [nvim-dap-python](https://github.com/mfussenegger/nvim-dap-python)
  - [nvim-dap-ui](https://github.com/rcarriga/nvim-dap-ui)
- then configuring the plugins

## Enable DAP in Lunarvim

Set `lvim.builtin.dap.active = true` in `~/.config/lvim/config.lua`

## Add the relevant plugins

These could go anywhere. E.g. in the main `~.config/lvim/config.lua`.
They or they could go in `~/.config/lvim/lua/user/plugins.lua` and be called by `require("user.plugins").config()` from `~/.config/lvim/config.lua`.

    {
      "mfussenegger/nvim-dap-python",
      config = function()
        require('dap-python').setup('~/.virtualenvs/debugpy/bin/python')
        require('dap.ext.vscode').load_launchjs()
      end,
    },
    { "leoluz/nvim-dap-go", module = "dap-go" },
    {
      "rcarriga/nvim-dap-ui",
      config = function()
        require("dapui").setup()
      end,
      ft = { "python", "go" },
      event = "BufReadPost",
      requires = { "mfussenegger/nvim-dap" },
      disable = not lvim.builtin.dap.active,
    },

## Configure the plugins

The obvious part here is to read the directions for `nvim-dap`, `nvim-dap-python`, and `nvim-dap-ui`.

#### `nvim-dap` and `nvim-dap-python`

`nvim-dap` needs adapters to run. E.g. for Python, it needs `debugpy` both locally, and in a Docker environment if debugging in Docker. See the [debug-adapters](https://github.com/mfussenegger/nvim-dap/wiki/Debug-Adapter-installation#python) section of the `nvim-dap` wiki, in conjunction with the [nvim-dap-python](https://github.com/mfussenegger/nvim-dap-python) documentation, but the short of it is that there should be a virtual environment just for `debugpy`

    mkdir .virtualenvs
    cd .virtualenvs
    python -m venv debugpy
    debugpy/bin/python -m pip install debugpy

This is then called as part of the plugin setup up above: `require('dap-python').setup('~/.virtualenvs/debugpy/bin/python')`

#### `.vscode/launch.json`

`nvim-dap` can load project details from a custom `launch.json`, rather than via edits to `dap.configurations.<language>`. See Open Library's [launch.json](https://github.com/internetarchive/openlibrary/blob/6d10bb3cf441a2aed594f018fb150c4f477630f1/.vscode/launch.json) for an example.

The short of it is that the `type` specifies the language to modify within dap: e.g. `type: python` is the same as `dap.configurations.python`.

See the [Custom configuration](https://github.com/mfussenegger/nvim-dap-python#custom-configuration) section of [vim-dap-python](https://github.com/mfussenegger/nvim-dap-python) for more.

## Docker

The local development environment for Open Library already does this. See the [docker-compose.override.yml](https://github.com/internetarchive/openlibrary/blob/c0694cc6ad846b8b684edf78a4b087f64eecdf7e/docker-compose.override.yml) as an example. But also see the Open Library specific notes below.

## Doing the debugging

Again, this is specific to Open Library. To start `debugpy` on the correct process, first follow the [directions](https://github.com/internetarchive/openlibrary/wiki/Debugging), specifically about changing the number of workers.
Then visit [/admin/attach_debugger](http://localhost:8080/admin/attach_debugger) and click "Start". This starts `debugpy` on port 3000, and Docker is already configured to forward that port.

In neovim, <leader>d, and select 'start', and `Python: Attach`. This name comes from `.vscode/launch.json`. Do not use the remote connection option because it won't have the local and remote directories configured; `.vscode/launch.json` takes care of this.

## Open Library specific changes

Rather than change the number of works in the default `docker-compose.yml`, it's possible to export `$GUNICORN_OPTS` to `--reload --workers 1 --timeout 180` in `.env` or something.

It's also possible to have the debugger running any time `$DEBUG=True`, though this requires changing three files that can't be committed back to main. The idea is to add an environment variable, `$DEBUG`, such that if it's `True`, the server will run `debugpy` on port 3000:
`docker-compose.yml`, around lines 7-8 or so, under `environment:`:

    environment:
      - OL_CONFIG=${OL_CONFIG:-/openlibrary/conf/openlibrary.yml}
      - GUNICORN_OPTS=${GUNICORN_OPTS:- --reload --workers 4 --timeout 180}
      - DEBUG=${DEBUG:- False}  # Add this

Add a `.env` in the openlibrary root for `docker-compose` to pull environment variables from. Inside, add:

    DEBUG=True
    GUNICORN_OPTS=--reload --workers 1 --timeout 180

This will override the default vales for `$DEBUG` and `$GUNICORN_OPTS`. `DEBUG=True` will be read by a function we'll add to `openlibrary/plugins/admin/code.py`, and the override for `$GUNICORN_OPTS` changes the number of workers so `debugpy` can attach to the one worker process.

Edit `openlibrary/plugins/admin/code.py` to add, around line 919, just above `setup()` at the very end:

    # Debug
    if os.getenv("DEBUG"):
        import debugpy
        debugpy.listen(address=('0.0.0.0', 3000))
        logger.info("Debugger listening on port 3000")

Now the debugger will be running every time `gunincorn` restarts (i.e. every save), and it can be connected to, within Lunarvim anyway, with `<leader>ds` and selecting `Python: Attach`, as specified in `.vscode/launch.json` above.
