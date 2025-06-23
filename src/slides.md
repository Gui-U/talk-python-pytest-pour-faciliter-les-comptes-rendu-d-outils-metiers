---
title: "Pytest pour faciliter les comptes rendu d'outils métiers"
---

# pytest pour faciliter les comptes rendu d'outils métiers
### Lighting talk rapide
24 juin 2025 - Meetup Python Grenoble - [turbine.coop](https://turbine.coop/)

---

# Quelques mots sur le speaker...

- Ingénieur AI/Devops chez Asygn depuis 6 ans
- Softeux incompris dans une équipe de hard

---

# Contexte
- Execution séquentielle des tests : si le premier fail, les autres ne sont pas execéutés
- De vieux outils avec des sorties diverses
- Peu lisible lors de l'on-boarding 
- Execution des "simulations top" systématiquement (avant livraison)

---
# L'utilisation de Git et de la CI dans cette équipe

- Git en trunk-based
![truck based illustration](trunck.png)
<!-- https://posthog.com/product-engineers/trunk-based-development -->

---
- La CI peut rester cassée plusieurs semaines

Bonus : push avant de partir en vacances
<!-- .element: class="fragment" -->

---

## Une non reg obscure à lancer...
<style>
.reveal pre code {
  max-height: 90vh;
  overflow: auto;
</style>

```bash [|1,3|5|7,11|16]
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
        if [ -d "$config_path" ]; then
        config_name=$(basename $config_path)
        echo "Testing" $test_name "config" $config_name "..."
        output_filename="../../${test_name}_${config_name}_test.txt"
        simu.csh -simulator QUESTA -config ../CONFIG/$config_path -testcase all -nopopup -nowave > $output_filename
        fi
    done
    cd ../..
done
```
---

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
---

# Constat
- De plus en plus difficile d'avoir une vue d'ensemble
- Un test cassé peut en cacher un autre
- Mal intégrable en CI

---

# Pourquoi pytest ?

- Solution éprouvée dans le monde python
- Facilement hackable
- Maitrisé par d'autres équipes

# Diverses itérations
## Agrégation
## Ajout du temps
## Utilisation des outputs htmls (pages) et junit

---

# Résultat

![pytest_html_white](pytest_html_white.png)

---

# Outputs

- .junit
- html

---

# Résultats

---
## Points positifs

- Lecture en cas de problèmes, permet de faire de l'archéologie
- Permet aux softeux de comprendre pourquoi leur code ne fonctionne pas (c'est le hardware qui est cassé)
---
## Points négatifs

- CI constament cassée à cause du "trunck based"
- Pas un réflexe pour les devs hardware
- Lecture uniquement en cas de manque de temps
- Robustesse moyenne

---

## Points négatifs
Exemple : 2 dégradations coup sur coup !

![dégradation](degradation.png)

---
