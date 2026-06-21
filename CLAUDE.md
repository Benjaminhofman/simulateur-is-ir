# CLAUDE.md — simulateur-is-ir

## Présentation
Simulateur de comparaison fiscale EI/IR vs EURL/IS pour dirigeants TPE/PME.
Fichier unique : `index.html` — HTML/JS statique, aucune dépendance backend.
Déployé sur Netlify : https://silver-stardust-481a03.netlify.app
Repo GitHub : Benjaminhofman/simulateur-is-ir (branche main)

---

## Conventions de travail

- Toujours répondre en français
- Prompts courts, séquentiels, jamais de confirmation avant d'exécuter
- Toujours committer et pusher après chaque modification
- Netlify auto-deploy sur push GitHub → pas d'action manuelle requise

---

## Stack technique

- HTML/JS statique — fichier unique index.html
- Chart.js CDN (graphique courbes)
- Google Fonts : DM Mono + Syne + Inter
- Aucun framework, aucun build step
- Netlify (publish directory : `.`)

---

## Règles fiscales 2026 intégrées

### Réforme assiette TNS (avril 2026)
- Assiette sociale = revenu brut × 74% (abattement forfaitaire 26%)
- Plancher abattement : 845.86 €
- Plafond abattement : 62 478 €
- CSG-CRDS calculée sur même assiette unique (fin du calcul circulaire)
- Base IR = revenu net + CSG non déductible (2.9% × assiette brute)

### PASS 2026
- PASS = 46 368 €

### Cotisations TNS par postes (taux réels 2026)
```
Maladie :
  < 40% PASS  : 0.5%
  < 110% PASS : 5%
  > 110% PASS : 8.5%  (taux plein réforme 2026)

Retraite base :
  T1 (≤ PASS) : 17.75%
  T2 (> PASS) : 0.72%

Retraite complémentaire :
  T1 (≤ PASS)     : 8.1%
  T2 (1–4 PASS)   : 9.1%

Invalidité-décès : 1.3% (plafonné à 1 PASS)

Allocations familiales :
  < 110% PASS : 0%
  110–140% PASS : 1.7%
  > 140% PASS : 3.1%

CFP : forfait 116 €

CSG-CRDS : 9.7% sur assiette unique (réforme : plus de majoration cotisations)
```

### IS
- Taux réduit : 15% jusqu'à 42 500 € de résultat fiscal
- Taux normal : 25% au-delà

### Barème IR 2026
```
0%  jusqu'à 11 600 €
11% de 11 601 à 29 579 €
30% de 29 580 à 84 577 €
41% de 84 578 à 181 917 €
45% au-delà de 181 917 €
```
Validé au centime près contre publicodes/modele-ti (modèle officiel URSSAF).

---

## Architecture du calcul

### Variable d'entrée principale
L'utilisateur saisit le **revenu net annuel (avant IR)**.
Le simulateur reconstitue le CA brut par itération (point fixe, 10 passes) :
```js
function reconstitueBrut(net) {
  let brut = net * 1.45;
  for (let i = 0; i < 10; i++) {
    const cotis = calcCotisTNS(calcAbattement(brut));
    brut = net + cotis;
  }
  return brut;
}
```

### calcAbattement(revenu)
```js
const abattement_brut = revenu * 0.26;
const abattement = Math.max(845.86, Math.min(abattement_brut, 62478));
return revenu - abattement;  // assiette sociale
```

### calcCotisTNS(assiette)
Calcul par postes selon les taux 2026 ci-dessus.
CSG = assiette × 9.7% (assiette unique, sans majoration cotisations).

### calcIR(revenu_imposable)
Barème progressif 2026 sur 1 part fiscale.

### EI/IR
```
brut = reconstitueBrut(revenu_net)
cotisations = calcCotisTNS(calcAbattement(brut))
csg_non_ded = calcAbattement(brut) × 2.9%
base_ir = revenu_net + csg_non_ded
ir = calcIR(base_ir)
cout_total = cotisations + ir
net_entreprise = 0
```

### EURL/IS
```
remun_nette = revenu_net × (distrib / 100)
brut_remun = reconstitueBrut(remun_nette)
cotisations = calcCotisTNS(calcAbattement(brut_remun))
resultat_is = brut_CA - brut_remun
is_du = min(resultat_is, 42500) × 15% + max(0, resultat_is - 42500) × 25%
net_entreprise = resultat_is - is_du
csg_non_ded = calcAbattement(brut_remun) × 2.9%
base_ir = remun_nette + csg_non_ded
ir = calcIR(base_ir)
cout_total = cotisations + is_du + ir
```

---

## Enseignements clés (validés mathématiquement)

- À distribution 100% : EI/IR = EURL/IS (IS = 0, mêmes cotisations)
- L'avantage EURL/IS n'apparaît que quand le dirigeant garde de l'argent en société
- Si tout sorti en salaire : EURL/IS coûte plus cher (IS + cotis + IR > cotis + IR direct)
- L'avantage IS = capitalisation à 15% vs cotisations + IR immédiats en EI/IR
- Le seuil de bascule réel n'est pas un niveau de revenu mais un besoin de liquidité

---

## Affichage

### Panneau gauche (paramètres)
- Curseur revenu net annuel (15k–300k)
- CA reconstitué affiché en lecture seule
- Curseur part versée en rémunération (10–100%)
- Select TMI (conservé pour référence, non utilisé dans calcul principal)

### Cases verdict (2 colonnes)
- cout_total en grand
- taux_effectif = cout_total / brut × 100
- Badge "✓ Plus avantageux" sur le gagnant
- Encadré "Économie Xk€/an" entre les deux cases

### Tableau décomposition
Colonnes : Charges sociales / Impôt société / Impôt revenu / Total prélevé / Taux réel / Réserve société

### Encadré pédagogique (dynamique)
Montants mis à jour à chaque update() :
- revenu net EI/IR = revenu_net saisi
- rémunération EURL/IS = remun_nette
- réserve société = net_entreprise

### Graphique
Chart.js — courbes coût total selon le revenu brut (15k–300k, pas 5k)
Ligne verticale sur le revenu saisi.

---

## Déploiement

- Netlify : auto-deploy sur push branche main
- Aucune variable d'environnement
- Publish directory : `.`
- Build command : aucune

## Historique déploiement
- Render Static Site abandonné (quota minutes épuisé en juin 2026)
- Migration Netlify le 21 juin 2026

---

## PENDING

- [ ] Décote IR pour bas revenus (< 22k€ IR brut, mécanisme réducteur)
- [ ] Ajouter card sur fiscalite-entrepreneur.fr
- [ ] Renommer URL Netlify (simulateur-is-ir.netlify.app)
- [ ] Article de blog associé sur fiscalite-entrepreneur.fr
- [ ] CEHR (Contribution Exceptionnelle Hauts Revenus) pour revenus > 250k€
