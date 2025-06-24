---
title: "Pytest pour faciliter les comptes rendu d'outils métiers"
---
<style>
.reveal pre code {
  max-height: 95vh;
  overflow: auto;
</style>

# pytest pour faciliter les comptes rendus d'outils métiers
### Lighting talk

24 juin 2025 - [Meetup Python Grenoble](https://meetup-python-grenoble.github.io/) - Présenté à  [laturbine.coop](https://turbine.coop/)

[Source des slides sur Github](https://github.com/Gui-U/talk-python-pytest-pour-faciliter-les-comptes-rendu-d-outils-metiers)

---

## Quelques mots sur le speaker...

Guillaume Urard

- Ingénieur AI/Devops depuis 2019
- Domaines : Embarqué -> IA -> sysadmin -> devops -> embarqué
- Asygn : PME d'électronique
- Softeux incompris dans une équipe de hardware<!-- .element: class="fragment" -->

---

# Contexte du métier
- Des tests de non-régression sur du code de description matériel (VHDL/Verilog) existent
    - Lancés par des outils métiers propriétaires<!-- .element: class="fragment" -->
    - Exécution séquentielle : si le premier casse, les autres ne sont pas exécutés<!-- .element: class="fragment" -->
    - Sorties diverses et non standardisées<!-- .element: class="fragment" -->
- Flow peu lisible pour les nouveaux arrivants<!-- .element: class="fragment" -->
- Blocs indépendants / bloc de circuit électronique (IP)<!-- .element: class="fragment" -->
    - Parfois pas si indépendants<!-- .element: class="fragment" -->
    - "simulations top" nécessaires systématiquement (et pas qu'avant livraison)<!-- .element: class="fragment" -->

-v-

## L'utilisation de Git et de la CI dans ce projet

- Git en trunk-based
- Mono repository<!-- .element: class="fragment" -->
- La CI peut rester cassée plusieurs semaines <!-- .element: class="fragment" -->

Bonus : _git push_ vendredi soir avant de partir en vacances 
<!-- .element: class="fragment" -->

![[trunk based illustration](https://posthog.com/product-engineers/trunk-based-development)](trunk.png) <!--  .element: max-height="60vh" -->


-v-

## Contraintes

- Les tests sont déjà écrits et génèrent des fichiers de sortie texte
- On veut réutiliser les sorties avec un framework de test habituel<!-- .element: class="fragment" -->
- Ce framework n'a pas à lancer les tests, il a juste à lire les résultats des tests<!-- .element: class="fragment" -->
- Les résultats des tests contiennent des mots clés à détecter et parser<!-- .element: class="fragment" -->

-v-

## Des tests de non-régression obscurs à lancer...

Exemple avec un outil métier

```bash [|1,3|5|7,11|15]
source SETUP/global_setup
cd DESIGN
make
# Run simulations for all tests (IPs with MMAP directory)
export REGMAP_GENERATOR_TEST_DIRS="CACHE/MMAP I2C/MMAP IIR_ACCEL/MMAP SPI/MMAP TIMER/MMAP TIMER_SLOW/MMAP"

for dir in ${REGMAP_GENERATOR_TEST_DIRS}; do
    cd ${dir}/..
    test_name=$(basename $(pwd))
    cd MMAP
    for config_path in ../CONFIG/RTL* ../CONFIG/FPGA*; do
        config_name=$(basename $config_path)
        echo "Testing" $test_name "config" $config_name "..."
        output_filename="../../${test_name}_${config_name}_test.txt"
        simu.csh -simulator QUESTA -config ../CONFIG/$config_path -testcase all -nopopup -nowave > $output_filename
    done
    cd ../..
done
```

_(temps d'exécution non mesuré par mesure de simplicité)_

-v-

## ... et à comprendre

``` [|12]
$ tree -h
.
├── [ 22K]  CACHE_FPGA_test.txt
├── [8.4K]  CACHE_RTL_DPRAM_test.txt
├── [8.0K]  CACHE_RTL_SPRAM_test.txt
├── [ 74K]  I2C_FPGA_test.txt
├── [ 75K]  I2C_RTL_test.txt
├── [103K]  IIR_ACCEL_FPGA_test.txt
├── [6.4K]  IIR_ACCEL_RTL_test.txt
├── [1.0K]  SPI_REG_FPGA_test.txt
├── [1.0K]  SPI_REG_RTL_test.txt
├── [ 31M]  SPI_RTL_test.txt
├── [6.8K]  TIMER_FPGA_test.txt
├── [7.0K]  TIMER_RTL_test.txt
└── [7.5K]  TIMER_SLOW_RTL_test.txt
```


-v-

## Constat

- De plus en plus difficile d'avoir une vue d'ensemble<!-- .element: class="fragment" -->
- Certains tests sont rarement executés car trop longs pour du local<!-- .element: class="fragment" -->
- Mal intégrable en CI, sorties peu lisibles<!-- .element: class="fragment" -->

---

# Pourquoi pytest ?

- Solution éprouvée dans le monde python<!-- .element: class="fragment" -->
- Nombreux plugins<!-- .element: class="fragment" -->
- Facilement personnalisable<!-- .element: class="fragment" -->
- Divers formats de sortie<!-- .element: class="fragment" -->
  - HTML (plugin pytest-html)<!-- .element: class="fragment" -->
  - Junit<!-- .element: class="fragment" -->
- Maitrisé par d'autres équipes<!-- .element: class="fragment" -->

---

# Implémentations et itérations

Rapidement...

-v-

## Agrégation

```python [|9-10|8-10|4-6|12-22]
# read_test.py
import os, pytest

def log_files():
    log_files = [f for f in os.listdir(os.path.dirname(__file__)) if f.endswith("_test.txt")]
    return [os.path.join(os.path.dirname(__file__), f) for f in log_files]

@pytest.mark.parametrize("filename", log_files())
def test_read_file(filename):
    assert please_ignore_this_line_and_previous_one(filename)

# the name of function will be displayed in html result with pytest --tb=no
def please_ignore_this_line_and_previous_one(filename):
    test_logs = ""
    assert os.path.exists(filename)
    with open(filename, "r") as f:
        print(f.read())  # print all file content, including ###TESTDURATION string
    with open(filename, "r") as f:
        for line in f:
            if line.startswith("###"):
                test_logs += line
    return "###TESTRESULT=SUCCESS" in test_logs
```

-v-

## Ajout du temps et meilleurs noms de test

```python [|11,12,18]
# conftest.py
import os, re, pytest

def _extract_duration_from_log(logs):
    match = re.search(r"###TESTDURATION=([\d.]+)", logs)
    if match:
        return float(match.group(1))
    return 0.00

### Change pytest test duration
@pytest.hookimpl(tryfirst=True)
def pytest_report_teststatus(report, config):
    """This hook is called every time"""
    if report.when == "call":
        report.duration = _extract_duration_from_log(report.capstdout)

### Change pytest tests names
def pytest_itemcollected(item):
    """change test name, using fixture names"""
    filename = item.callspec.params["filename"]
    item._nodeid = os.path.basename(filename).replace("_test.txt", "").replace("_", " ")
```

---

## Résultat : html

![pytest_html_white](pytest_html_white.png)

-v-

## Résultat : junit

![junit_pipeline](junit_pipeline.png)
![junit_mr](junit_mr.png)<!-- .element: class="fragment" -->
<!-- .element: class="r-stack" -->

---

# Bilan

-v-

## Points positifs

- Lecture en cas de problèmes, permet de faire de l'archéologie (`git bisect`)
<!-- .element: class="fragment" -->

- Permet aux softeux de comprendre pourquoi leur code ne fonctionne pas (c'est le hardware qui est cassé)<!-- .element: class="fragment" -->

- Mesure le temps de chaque test<!-- .element: class="fragment" -->

- Principe copié pour les tests de compilation `C`, qui sont à base de Makefile
<!-- .element: class="fragment" -->

-v-
## Points négatifs

- Lecture qui n'est pas un réflexe pour les devs hardware, qui préfèrent les tests locaux
- Robustesse moyenne <!-- .element: class="fragment" -->
- CI constamment cassée à cause du "trunk based"  <!-- .element: class="fragment" -->
  - Mails de Gitlab CI déclarés en spam ! <!-- .element: class="fragment" -->

-v-

Exemple : 2 dégradations coup sur coup !

_Commit le plus récent_

![dégradation](degradation.png)

_Commit le plus vieux_

---

# Merci

-v-
