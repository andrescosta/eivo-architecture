Analyse
-------

Concepts:
- Professeurs
- Élèves 
- Classes
  - Groupes
- Topique 
  - Nom
- Activités
  - Avec/Sans qualification
  - Historique des transactions
  - Types
    - Conjugation
    - apportez tes propres verbes
  - Plugiciels
  - Adaptabilité
  - Historique


Fonctionnalité:
- Joues 
  - Activites
  - Individuels/groupals/classes


- Infra
    - Database
    - Services
    - React
    - Identity
    - CICD

Design
------

Database:
  Données:
    Utilisateurs
    Rôles
    Permissions
    Groupes
      types
    Activités
      types (IA, Humain, etc.)
    Plugiciels Métadonnée
    Historique

  Données transactionnelles et temporels:
    Jeux actifs
      Activités
      Participants
    Groupes temporel

  
API: /api/v1

/utilisateurs
/domaines
/classes
/membres
/activites

/jeux
/jeux/{id}

Conception de base de données
-----------------------------

Données:
locataires(id, nom(pas nul))
utilisateurs(id, locataire_id(pas nul), nom(pas nul), ext_id(pas nul))
domaines(id, nom(pas nul), locataire_id(pas nul), topique(nul), type(nul))
classes (id, domaine_id(pas nul), nom(pas nul), topiques[](pas nul))
membres(id, utilisateur_id, classe_id, role)
historique(id, membre_id(pas nul), date(pas nul), note(pas nul), temps_ecoule(pas nul))
activites(id, classe_id(pas nul), application_id(pas nul))

Métadonnée:
topiques(id, nom(pas nul), descriptions(nul))
applications(id, nom(pas nul), description(nul), p_image(nul), g_image(nul), topiques[](pas nul), c_type([humain|IA]))

Données transactionnelles et temporels:
jeux(id, nom(pas nul), description(nul), membres_ids(pas nul)[], date(pas nul), activite_id(pas nul))


- Tool to feed the database with the exercises
- API in data to get the exercises



GET
/lessons?appId=?
[
{
  id: 1,
  description: "aaaaaaaaaaaa",
  exercises: {
    [
      {
        id:1,
        descriptions: {
          kind: ""
          long: "",
          short: ""
        },
        variation:
          - group: 1
      },
      {
        id:1,
        descriptions: {
          kind: ""
          long: "",
          short: ""
        },
        variation:
          - group: 2
      },
      {
        descriptions: {
          kind: ""
          long: "",
          short: ""
        },
        variation:
          - group: 3
      }
    ]
  }
}
]