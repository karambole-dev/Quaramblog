#### Choisire deux nombres premiers p et q
```
p = 11
q = 13

*Qui ne se divise que par 1 et par lui même.*
```

#### Calculer N
```
N = p * q
N = 11 * 13
N = 143
```

#### Calculer phi(N)
"Combien de nombre plus petit que ``N`` ne partagent aucun diviseur commun avec ``N`` ?"

Dans la mesure ou ``N`` est la multiplication de deux nombres premiers on peut faire :
```
phi(N) = (p - 1) * (q - 1)
phi(N) = (11 - 1) * (13 - 1)
phi(N) = 10 * 12
phi(N) = 120
```

#### Trouver e

Telle que ``e`` premier avec ``phi(N)`` ; ``e`` et ``phi(N)`` ne doivent avoir aucun diviseur en commun à part 1.
```
2 divisible par 2 et 120 aussi | NO
3 divisible par 3 et 120 aussi | NO
7 divisible par 1 et 7 et 120 n'est pas divisible par 7 | OK
```

---

```
p = 11
q = 13
N = 143
phi(N) = 120
e = 7

clé publique = (N, e) = (143, 7)
```

---

#### Trouver d
Telle que : ``(e * d) % phi(N) = 1``
```
(7 * d) % 120 = 1
```

Pour faire ça on va tester les multiples de 120 + 1 :
```
(5 x 120) + 1 = 601
    601 / 7 = 85,85 | NO

(6 x 120) + 1 = 721
    721 / 7 = 103 | OK

(7 * 103) % 120 = 1
```

---

```
d = 103

clé privée = (N, d) = (143, 103)
```

---

#### Pourquoi ça marche ?

Un attaquant connait ``N`` et ``e`` mais pour récupérer ``d`` il faudrait qu'il récupère ``phi(N)`` mais il n'a pas ``p`` et ``q``.
Sant ``p`` et ``q`` récupérer combien de nombres plus petit que ``N`` ne partage aucun diviseur avec N prendrait trop de temps. La raison pour laquel on arrive nous à récupérer ``phi(N)`` c'est parce que ``N`` est le résultat de ``p*q`` et qu'on peu utilisé la technique du ``(p - 1) * (q - 1)``.

#### Chiffrer

```
M = 2 ; le message
```
```
C = 2 ** e % N
C = 2 ** 7 % 143
C = 128 % 143 (0 et reste 128)
C = 128
```

#### Dechiffrer

```
M = C ** d % N
M = 128 ** 103 % 143
```

```
$ python3
>>> 128 ** 103 % 143
2

M = 2
```

128 puissance 103 le résulta est très grand. Alors pour le calculer, au lieu de faire ``128*128*128 etc..``, les ordinateurs utilise la technique de l'exponentiation modulaire : exemple avec 4 puissance 8 mod 10
```
4 ** 2 = 16
4 ** 4 = (4 ** 2) ** 2 = 16 ** 2 = 256
256 % 10 = 6

4 ** 8 (4 ** 4) ** 2 = ici pas besoin de calculer le résultat, comme on sait déja que 4 ** 4 mod 10 = 6, on peut faire 6 ** 2 = 36
36 % 10 = 6
```

Cette technique permet à l'ordinateur de paralléliser le calcule.
