# Cas d'étude : Classer des amendements comme sourcés/non-sourcés avec ActiveTigger

L’objectif est de prendre en main le logiciel ActiveTigger à partir d’une étude de cas simple : détecter automatiquement les amendements dits “sourcés” au sein d’un corpus d’amendements déposés par les députées [féminin universel] de l’Assemblée Nationale entre 2024 et 2026. 

Public visé : débutant (la question de recherche est simple) et francophone (corpus en français). 

## 1. Le contexte

Les députées disposent du droit d’amendement, c’est-à-dire qu’elles peuvent soumettre au vote des modifications aux textes de loi qui leur sont présentés, que ce soit par le gouvernement (projet de loi) ou par d'autres députés (proposition de loi).   

Un amendement peut émerger de multiples interactions, allant de discussions avec des administrées jusqu’à la lecture de rapports experts. Cependant, certains sont travaillés en étroite collaboration avec un ou plusieurs groupe(s) d’intérêt, voire proposés “clé en main” par ceux-ci. Pour lutter contre le risque de conflits d’intérêt et promouvoir la transparence de leurs pratiques, certaines députées ont pris pour habitude de citer nommément l’origine de l’amendement lorsque celui-ci provient d’une collaboration avec un représentant d’intérêt. C’est ce qu’on appelle le **_sourcing des amendements_**. 

## 2. Question de recherche 

Ce cas d'étude a une visée pédagogique, mais il vise à étudier cette pratique. 
- La pratique est-elle répandue :
    - (RQ1) Combien d’amendements sont dits sourcés parmi les amendements déposés sous la XVIIème législature de l'Assemblée Nationale ? 
- La pratique est-elle _uniformément_ répandue :
    - (RQ2) Quel parti politique source-t-il le plus ses amendements ? 
    - (RQ3) Le volume d'amendements sourcés évolue-t-il au cours du temps ?
  
Méthodologiquement : Peut-on identifier les amendements sourcés sur un large corpus à partir d'un nombre limité d'annotations ? 

## 3. Données

On utilise ici le jeu de données des amendements de la XVIIème législature de l’Assemblée Nationale, disponible de façon ouverte sur le site data de l’Assemblée. Il contient 123,224 amendements, datés du 18-09-2024 au 21-07-2026. 

Après plusieurs étapes d’agrégation et de nettoyage des données (décrites dans dataset_preprocessing.ipynb), on travaille à partir d’un jeu de 65,388 amendements uniques (sans duplicats exacts), d’une longueur maximale de 1024 tokens. 

> **Échelle d'annotation:** On choisit d’annoter à l’échelle de l’amendement car l’échelle de la phrase nous conduit à une prévalence trop faible de la catégorie “source”. À l'échelle de la phrase, la prévalence est de 0,5-1% contre 8,5-9% à l'échelle de l'amendement. Ce choix est rendu possible car les amendements sont relativement courts et ont une cohérence thématique. 

Le **projet** alloue 10,000 données au train set, 300 au validation set et 300 au test set. Cela permet d’avoir ~25 cas sourcés dans les valid et test sets, ce qui permet de calculer des métriques de performance basiques. 

## 4. Annotation : itérations pour la fixation du codebook
Quatre étapes majeures d'itération du codebook, affiné grâce à l'annotation par active learning.  

1. Commencer par une définition naïve 
> Un amendement est sourcé quand il a été travaillé avec un représentant d’intérêt et que son origine est exprimée explicitement comme telle. 
   
Problème 1 : Des cas comme "Cet amendement a été suggéré par" ne sont pas contenus dans la définition. "Suggéré" n'est pas tout à fait "travaillé". 

2. Répondre au problème 1 en précisant la nature de l'infuence directe
> Un amendement est source quand il a été **suggéré par** ou travaillé avec un représentant d’intérêt et que son origine est exprimée explicitement comme telle. 

Problème 2 : Des cas comme "Cet amendement est issu des travaux du Sénat (...)" ou "Cet amendement a été travaillé avec l'Association des Maires de France" citent explicitement leur source mais peuvent être ambigus quant à l'_influence_ dont ils rendent compte. Des choix doivent être faits sur la notion de "source", ce qui nous pousse à réinterroger notre RQ :    
- - Cherche-t-on à dégager les amendements qui citent explicitement leur source ou spécificiquement ceux qui citent l'influence de groupes d'intérêts ? La démarche initiale portait sur la deuxième proposition, on choisit de rester dans cette direction.
- - Les associations d'élus sont-elles des "lobbies" ? Plus difficile à déterminer. Des recherches plus approfondies sur la notion de représentant d'intérêt sont nécessaires.    

3. Répondre au problème 2 en ajoutant une définition de la notion de représentant d'intérêt.   
> Ajout: On entend par "représentant d’intérêt" un groupe ou une institution extérieure à la représentation politique, représentant des intérêts privés, conformément à la définition de la HATVP. 

Problème 3 : Des cas comme "Cet amendement, travaillé avec la profession, (...)" ou "Cet amendement a été travaillé avec le collectif [xx]" n'entrent pas dans la définition de la HATVP. Cela peut paraître insatisfaisant : choix est fait de les y inclure.  

4. Répondre au problème 3 en élargissant la définition de la HATVP
> Ajout: Cette définition reprend celle de la HATVP mais élargit son champ en n'exigeant pas une personnalité morale du représentant d'intérêt. 

**La version finale complète du codebook peut être consultée dans codebook.md.** En plus de la définition de label discutée ici, elle contient des exemples de cas communs et limites. 

## 5. Modèles entrainés et modèle retenu
On essaie les modèles CamembertaV2base puis ModernCamemBERTbase car leurs caractéristiques correspondent aux données. Notamment, ils acceptent les context windows de 1024 tokens.   
Les essais sur Modern sont "pour voir" car V2 obtient déjà d'excellents scores.

Tous les modèles testés ont 1024 tokens de context window et les paramètres par défaut pour ce qui n'est pas précisé.

- Modèle A (Otter) : CamembertaV2base, 8 batch size, 2 gradient accumulation, 4 epochs. 
   - F1 Train-Eval: 0,922. Modèle peut encore apprendre.
   - F1 Valid: 0,965. 

- Modèle B (Hornet) : CamembertaV2base, 8 batch size, 2 gradient accumulation, 6 epochs.   
   - F1 Train-Eval: 0,96. 
   - F1 Valid: 1. Excellentes performances.

- Modèle C (Chinchilla) : CamembertaV2base, 8 batch size, 1 gradient accumulation, 6 epochs. 
   - F1 Train-Eval: 1. 
   - F1 Valid: 0,979. Très bon mais tend vers l'overfitting. 

- _Note : Si on essaie 16 batch size et 2 gradient accumulation, le GPU crash._ 

- Modèle D (Boxer) : ModernCamemBERTbase, 8 batch size, 2 gradient accumulation, 8 epochs.
   - F1 Train-Eval: 0,939. Eval-loss a une hausse abrupte, et ne descend pas très bas.    
   - F1 Valid: 0,959. 

- Modèle E (Husky) : ModernCamemBERTbase, 8 batch size, 1 gradient accumulation, 8 epochs.
   - F1 Train-Eval: 0,96. Mieux que le modèle D, Eval-loss a une baisse abrupte. 
   - F1 Valid: 0,959. 

La plupart des mauvaises prédictions concerne des cas difficiles : les associations d'élus ou les organes de ministères. Syntaxiquement, ils sont présentés de la même manière que les lobbies mais nous avons fait le choix de les définir comme non-sourcés dans le Codebook.

On sélectionne le modèle B car ses scores de performance sur le validation set sont les meilleurs (et ses courbes de loss ont les plus belles formes). 

## 6. Résultats

Les résultats sont calculés sur deux datasets (voir results.ipynb) : 1) le dataset des expose_sommaire uniques (celui qu'on a importé dans ActiveTigger), et 2) l'extension de ces prédictions au dataset entier, à l'exclusion des amendements dont l'expose_sommaire est null et dont la longueur est supérieure à 1024 tokens. 

Les différences entre les deux sets sont maigres.     
On commente ici les résultats sur le dataset complet.

- (RQ1) : 6,6% des amendements seraient sourcés. La pratique est minoritaire mais pas non plus marginale.      
- (RQ2) : En proportion, les non inscrits sont ceux qui déposent le plus d'amendements sourcés (17,51% de leurs amendements déposés sont sourcés). Sont-ils davantage soumis à influence ou davantage transparents ? Il faut aussi mettre en perspective ces chiffres avec le volume d'amendements déposés : les non-inscrits n'ont déposé que 695 amendements sur la période, contre 18k pour LFI.  
- (RQ3) : Le volume d'amendements sourcé varie fortement au cours du temps. Les grands moments parlementaires (projets loi finance) et des textes de loi concernant l'agriculture (proposition de loi visant à lever les contraintes à l'exercice du métier d'agriculteur dite "Loi Duplomb",projet de loi d'urgence pour la protection et la souveraineté agricoles) correspondent aux volumes d'amendements les plus élevés. L'importance de textes relatifs à l'agriculture pose de nouvelles question : les groupes d'intérêt sont-ils particulièrement actifs dans ce secteur ou bien leur influence est-elle davantage notifiée par rapport à d'autres secteurs ? Quels groupes d'intérêt sont les plus cités et par quels groupes politiques ?

RQ Méthodologique : Notre étude a été possible, et on obtient de très bons scores à partir d'un nombre très limité d'annotations. 

## 7. Ouverture : intérêt, limites

### 7.1. Est-ce qu'un decoder (Claude, ChatGPT) aurait pu le faire ? 
Oui, fort probablement, surtout si on avait disposé d'un bon codebook dès le départ.  Mais mettre en perspective la nature de notre cas avant d'en tirer des conclusions générales : 
- **Qualité de la classification**: Il s'agit d'un cas d'étude simple pour lequel les cas limites sont peu nombreux et facilement catégorisables, et dont la plupart des données ne demande pas un jugement subjectif de la phrase. Un decoder aurait probablement pu le faire avec la même qualité que notre encoder, mais sur d'autres types de tâches et données cela n'est pas garanti. 
- **Coût temporel** : L'utilisation d'un decoder reste plus rapide, mais étant donné le nombre limité d'annotations nécessaire ici, le fine-tuning du modèle n'a pas été extrêmement chronophrage.
- **Sensibilité des données** : Étant donné qu'il s'agit d'un cas d'étude, nous avons utilisé des données 100% ouvertes que nous aurions pu fournir à un decoder commercial. En situation réelle, il est plus rare de pouvoir le faire puisqu'aucune donnée sensible ne doit être fournie à ces outils.
- **Progression de la connaissance de la chercheuse** : On ne disposait pas d'un codebook précis avant de démarrer le fine-tuning. La confrontation aux données permettent à la chercheuse de mieux connaître ses données, d'affiner son codebook, et de réinterroger ses questions et objectifs de recherche. C'est le propre de l'utilisation d'un encoder, qui est absente de la méthode par decoder.  

À cela s'ajoutent les arguments généraux :
- **Réplicabilité**: Utiliser des encoders open-weight permet une meilleure réplicabilité de l'étude. 
- **Coût économique** : Afin de faire réaliser cette tâche par un decoder commercial, un abonnement aurait été nécessaire. Pour l'encoder, l'accès à un GPU est nécessaire. 
- **Coût environnemental** : Modèles decoder plus larges donc plus coûteux. 

### 7.2. Ce que l'on aurait pu ajouter ou faire différemment
- Une exploration préalable des données via les outils intégrés dans ActiveTigger. 
- Plus d'annotations dans les validation et test sets afin d'améliorer la précision du F1 source. 
- Plus d'annotations du label "source" dans le train set pour équilibrer les catégories au sein du train set et affiner la compréhension des cas limites par le modèle. 
- Une annotation croisée d'une partie des données pour obtenir un interannotator agreement score. 
- Une exploration manuelle des 664 amendements uniques retirés car leur longueur excédait 1024 tokens. 

### 7.3. Ce que l'on pourrait faire ensuite
Cette prédiction pourrait ensuite servir à entraîner d'autres classifiers multilabels sur les amendements sourcés afin d'enquêter sur l'identité des représentants d'intérêt ou sur les types de texte les plus susceptibles de contenir du sourcing.
