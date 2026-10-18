# GitOps — Lab 6: Dynamic Generator

מימוש תרגיל GitOps באמצעות Argo CD ApplicationSet ו־Git Directory Generator.
המטרה היא להגדיר שתי אפליקציות ולבחור לפריסה רק את סביבת `qa`.
התרגיל משתמש בשתי האפליקציות המקוריות מתוך `manifests/`:

- `app1.yaml`: Application של Argo CD עבור `flask-aws-monitor`.
- `app2.yaml`: Deployment של NGINX בשם `my-nginx-app`, עם שני replicas.

**הכול הוכן מקומית בלבד. לא בוצעו commit, push, העלאה ל־GitHub או פריסה לקלאסטר.**

## מבנה התיקיות

```text
gitops/
├── README.md
├── manifests/
│   ├── app1.yaml
│   └── app2.yaml
├── applicationsets/
│   └── qa-applicationset.yaml
└── systems/
    ├── devops/
    │   ├── dev/
    │   │   └── ingress.yaml
    │   └── qa/
    │       └── ingress.yaml
    ├── team-a/
    │   ├── dev/
    │   │   └── app1.yaml
    │   ├── prd/
    │   │   └── app1.yaml
    │   └── qa/
    │       └── app1.yaml
    └── team-b/
        ├── dev/
        │   └── app2.yaml
        ├── prd/
        │   └── app2.yaml
        └── qa/
            └── app2.yaml
```

קובצי האפליקציות תחת `systems/` הם עותקים זהים לקבצים המקוריים ב־`manifests/`.
קובצי המקור נשארו ללא שינוי.

## קבצים מרכזיים

| קובץ | תפקיד |
| --- | --- |
| [manifests/app1.yaml](manifests/app1.yaml) | הגדרת Application המקורית של Flask |
| [manifests/app2.yaml](manifests/app2.yaml) | ה־Deployment המקורי של NGINX |
| [applicationsets/qa-applicationset.yaml](applicationsets/qa-applicationset.yaml) | הגילוי של תיקיות QA ויצירת שני ה־Applications |
| [systems/team-a/qa/app1.yaml](systems/team-a/qa/app1.yaml) | עותק הגדרת Flask לצורך מבנה התרגיל |
| [systems/team-b/qa/app2.yaml](systems/team-b/qa/app2.yaml) | ה־Deployment שממנו נפרס NGINX בסביבת QA |

## שתי אפליקציות QA בלבד

ב־ApplicationSet אחד מוגדרים שני Git Directory Generators:

| תיקייה שנבחרת | Application שנוצר | מקור הפריסה | Namespace |
| --- | --- | --- | --- |
| `systems/team-a/qa` | `flask-aws-monitor-qa` | הריפו והנתיב מתוך `manifests/app1.yaml` | `flask-aws-monitor-qa` |
| `systems/team-b/qa` | `my-nginx-app-qa` | `systems/team-b/qa` בריפו GitOps | `my-nginx-app-qa` |

רק תיקיות `qa` נבחרות. תיקיות `dev`,‏ `prd` ו־`devops` אינן נבחרות,
כך שה־ApplicationSet יוצר בדיוק שני Applications במבנה הנוכחי.
ה־generators מזהים את קיום התיקיות ב־Git ומשתמשים ב־`values` להעברת
פרטי המקור השונים אל ה־template המשותף.

הנתיבים מוגדרים במפורש כ־`systems/team-a/qa` ו־`systems/team-b/qa`.
הוספת צוות חדש לתיקיית `systems/` מחייבת הוספת generator מתאים;
ההגדרה הנוכחית אינה בוחרת תיקיות QA של צוותים נוספים אוטומטית.

`app1.yaml` הוא כבר Application ולא Deployment. כדי להימנע מיצירת
Application נוסף שמכיל בתוכו Application של Flask, ה־ApplicationSet
משתמש ישירות ב־`repoURL`, ב־`path` וב־`project` שלו:

```text
repoURL: https://github.com/DXNOAM/devops-course-app-June-2026.git
path: .
project: gitops-project
```

עותקי `app1.yaml` תחת `systems/team-a/` מתעדים את מבנה התרגיל ואינם נפרסים
כמשאבים מקוננים. שינוי בפרטי המקור או הפרויקט של Flask בעתיד דורש עדכון
גם בערכי ה־generator הראשון. בהיעדר revision בקובץ המקורי, Flask משתמש ב־`HEAD`.

NGINX נפרס מה־Deployment המקורי, כולל `nginx:latest`, שני replicas
והתווית `app: nginx`. ה־Application שלו משתמש בפרויקט המובנה `default`.
שמות ה־Applications וה־namespaces מקבלים את הסיומת `qa` לצורך הפרדה.
מדיניות הסנכרון כוללת:

- סנכרון אוטומטי של שינויים ממקור ה־Git של כל אפליקציה.
- `prune: true`: הסרת משאבים שהוסרו ממקור הפריסה.
- `selfHeal: true`: תיקון סטייה בין מצב הקלאסטר להגדרה ב־Git.
- `CreateNamespace=true`: יצירת namespace היעד בעת הצורך.

## Ingress

לשמירת מבנה התמונה, `systems/devops/{dev,qa}/ingress.yaml` כולל Ingress
ו־Service שמפנה ל־NGINX לפי התווית הקיימת `app: nginx`.
הוא אינו כולל Deployment נוסף ואינו נבחר ב־ApplicationSet.
אם משתמשים בו בנפרד בעתיד, יש לפרוס אותו באותו namespace של NGINX,
להתקין Ingress controller שתומך במחלקה `nginx` ולהגדיר את כתובת ה־host.
ניתן לבדוק את NGINX גם ללא Ingress או Service באמצעות port-forward ל־Deployment.

## דרישות להפעלה עתידית

נדרשים Argo CD עם ApplicationSet controller ב־namespace בשם `argocd`,
גישה לשני הריפוזיטוריז, ופרויקט קיים `gitops-project` שמורשה לקרוא את ריפו Flask
ולפרוס ל־namespace בשם `flask-aws-monitor-qa`.
גם פרויקט `default` צריך לאפשר את מקור GitOps ואת namespace היעד של NGINX.
משאבים בריפו Flask שמגדירים namespace מפורש עשויים לדרוש התאמה לסביבת QA;
תוכן הריפו החיצוני לא נבדק במסגרת ההכנה המקומית.

Argo CD קורא מה־remote ואינו רואה שינויים מקומיים. ההגדרה מוכנה לבדיקה
מקומית, אך לא תופעל מול תוכן הריפו הקיים עד שתיקיות התרגיל יהיו זמינות בו.
אין לבצע העלאה או פריסה במסגרת ההכנה הנוכחית.

לאחר החלטה מפורשת להפעיל ובהתקיים הדרישות, פקודת ההתקנה מתוך שורש הריפו היא:

```powershell
kubectl apply -f applicationsets/qa-applicationset.yaml
```

פקודות בדיקה לשימוש לאחר הפעלה:

```powershell
kubectl get applicationsets systems-qa -n argocd
kubectl get applications -n argocd -l environment=qa
kubectl get deployments -n flask-aws-monitor-qa
kubectl get deployment my-nginx-app -n my-nginx-app-qa
kubectl port-forward -n my-nginx-app-qa deployment/my-nginx-app 8080:80
```

לאחר port-forward ניתן לפתוח `http://localhost:8080`.
אין להפעיל במקביל את `manifests/app1.yaml` או את העותקים שלו באופן ידני
לצד ה־ApplicationSet, כדי להימנע מפריסה נפרדת נוספת של Flask.

## בדיקות שבוצעו מקומית

- פענוח קובצי ה־YAML באמצעות `kubectl patch --local` ללא פנייה ל־API של קלאסטר.
- השוואת ששת עותקי האפליקציות תחת `systems/` לקובצי המקור: התוכן זהה.
- בדיקת בחירת הנתיבים והצבת פרמטרי ה־template המקומית: מתקבלות בדיוק
  `flask-aws-monitor-qa` ו־`my-nginx-app-qa`.
- בדיקת התאמה בין ה־Service האופציונלי לבין התוויות והפורט של NGINX.

הבדיקות אינן כוללות הפעלה של ApplicationSet controller, אימות מול סכמת CRD
או בדיקת זמינות של Flask בקלאסטר.

תיעוד רשמי: [Argo CD Git Directory Generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Git/).
