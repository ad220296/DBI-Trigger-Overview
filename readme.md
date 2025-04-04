
# 🔄 Vergleich & Beispiele: Trigger in PL/SQL

---

## 📘 Trigger in Oracle – Einleitung

Ein **Trigger** ist ein automatischer PL/SQL-Block, der bei bestimmten Ereignissen auf Datenbankobjekten wie Tabellen oder Views ausgeführt wird – z. B. bei `INSERT`, `UPDATE` oder `DELETE`.

Es gibt verschiedene Trigger-Arten:

| Typ               | Anwendung      | Besonderheit                                                                 |
|------------------|----------------|------------------------------------------------------------------------------|
| 🟡 Normal Trigger | Tabelle         | Einfacher Zeilen-Trigger, kein direkter Zugriff auf die Tabelle im Row-Level |
| 🔵 INSTEAD OF     | View            | Ersetzt INSERT, UPDATE oder DELETE auf einem View durch benutzerdefinierte Operationen |
| 🧩 COMPOUND       | Tabelle         | Kombiniert mehrere Trigger-Bereiche (Statement-Level & Row-Level) in einem Trigger |

---

## ⏰ Trigger-Zeitpunkte in PL/SQL: BEFORE, AFTER, INSTEAD OF

Trigger in PL/SQL können zu verschiedenen **Zeitpunkten** im Ablauf eines DML-Befehls ausgelöst werden. Diese bestimmen, **wann genau** der Trigger im Vergleich zur SQL-Aktion ausgeführt wird.

### 📌 Übersicht der Zeitpunkte

| Schlüsselwort     | Beschreibung                                                                 |
|-------------------|-------------------------------------------------------------------------------|
| `BEFORE`          | Wird **vor** dem Einfügen, Aktualisieren oder Löschen ausgeführt             |
| `AFTER`           | Wird **nach** dem Einfügen, Aktualisieren oder Löschen ausgeführt            |
| `INSTEAD OF`      | Wird **anstatt** eines DML-Befehls auf einem View ausgeführt                 |

👉 `BEFORE` eignet sich z. B. zur **Überprüfung oder Manipulation** von Daten, bevor sie gespeichert werden.  
👉 `AFTER` wird häufig zur **Protokollierung** oder für **Folgeaktionen** genutzt.  
👉 `INSTEAD OF` ist für **Views** nötig, da man auf Views nicht direkt schreiben kann.

---

## 🟡 1. Normaler BEFORE/AFTER Trigger (Tabelle)

Wird auf einer **Tabelle** definiert und reagiert auf INSERT, UPDATE oder DELETE.  
Verwendet `:NEW` und `:OLD` für jede betroffene Zeile.

```sql
CREATE OR REPLACE TRIGGER trg_check_sal
BEFORE INSERT OR UPDATE ON emp
FOR EACH ROW
BEGIN
    IF :NEW.sal < 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Gehalt darf nicht negativ sein!');
    END IF;
END;
/
```

---

## 🔵 2. INSTEAD OF Trigger (für Views)

Ein **INSTEAD OF Trigger** wird auf einem **View** definiert und ersetzt `INSERT`, `UPDATE` oder `DELETE`-Operationen, die auf dem View selbst nicht direkt möglich wären.

📌 Typisches Szenario:  
Ein View zeigt **Mitarbeiterdaten mit dem Namen der Abteilung**, statt mit der `DEPTNO`.  
Benutzer kennt nur `DNAME`, aber die Tabelle braucht `DEPTNO`.  
➡️ Der Trigger übernimmt die Zuordnung im Hintergrund automatisch.

```sql
CREATE OR REPLACE VIEW emp_with_dname AS
SELECT 
    e.empno,
    e.ename,
    e.sal,
    d.dname
FROM emp e
JOIN dept d ON e.deptno = d.deptno;
```

```sql
CREATE OR REPLACE TRIGGER trg_emp_with_dname
INSTEAD OF INSERT OR UPDATE ON emp_with_dname
FOR EACH ROW
DECLARE
    v_deptno dept.deptno%TYPE;
BEGIN
    SELECT deptno INTO v_deptno
    FROM dept
    WHERE dname = :NEW.dname;

    IF INSERTING THEN
        INSERT INTO emp (empno, ename, sal, deptno)
        VALUES (:NEW.empno, :NEW.ename, :NEW.sal, v_deptno);
    ELSIF UPDATING THEN
        UPDATE emp
        SET ename = :NEW.ename,
            sal = :NEW.sal,
            deptno = v_deptno
        WHERE empno = :OLD.empno;
    END IF;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20001, 'Abteilung nicht gefunden: ' || :NEW.dname);
END;
/
```

---

## 🧩 3. Compound Trigger – Mehrere Triggerbereiche in einem

Ein **Compound Trigger** kombiniert mehrere Triggerabschnitte in einem einzigen Objekt. Das ist besonders hilfreich, wenn du z. B. Daten aus der Tabelle brauchst, aber gleichzeitig auf einzelne Zeilen reagieren willst.

### 🔗 Triggerbereiche im Compound Trigger

| Abschnitt            | Wann er ausgeführt wird              | Typische Verwendung                                 |
|----------------------|--------------------------------------|-----------------------------------------------------|
| `BEFORE STATEMENT`   | **Einmal vor** der gesamten Aktion   | Maximalwerte, Initialisierungen, Vorbereitungen     |
| `BEFORE EACH ROW`    | Vor **jeder betroffenen Zeile**      | Validierungen, Anpassungen der Zeile                |
| `AFTER EACH ROW`     | Nach **jeder betroffenen Zeile**     | Protokollierung, Kettenreaktionen                   |
| `AFTER STATEMENT`    | **Einmal nach** der gesamten Aktion  | Aufräumen, Ausgaben, Gesamtberechnungen             |

> 🔄 So kannst du z. B. im `BEFORE STATEMENT` den höchsten Gehaltswert laden und ihn im `BEFORE EACH ROW` für jede eingefügte/aktualisierte Zeile als Grenze verwenden.

➡️ Ermöglicht z. B. Zugriff auf die Tabelle vor dem Zeilen-Trigger!

```sql
CREATE OR REPLACE TRIGGER trg_limit_salary
FOR INSERT OR UPDATE OF sal ON emp
COMPOUND TRIGGER

    max_sal NUMBER;

BEFORE STATEMENT IS
BEGIN
    SELECT sal INTO max_sal FROM emp WHERE ename = 'KING';
END BEFORE STATEMENT;

BEFORE EACH ROW IS
BEGIN
    IF :NEW.sal > max_sal AND :NEW.ename != 'KING' THEN
        :NEW.sal := max_sal;
    END IF;
END BEFORE EACH ROW;

END;
/
```

---

## 📊 Zusammenfassung der Trigger-Typen

| 🔍 Merkmal                              | 🟡 Normal Trigger | 🔵 INSTEAD OF Trigger | 🧩 COMPOUND Trigger |
|----------------------------------------|-------------------|-----------------------|---------------------|
| Gültig für                             | Tabelle           | View                  | Tabelle             |
| Zugriff auf Tabelle (SELECT etc.)      | ❌ (nur Statement) | ✅ indirekt            | ✅ über Statement   |
| Zugriff auf Zeilendaten (:NEW/:OLD)    | ✅                 | ✅                    | ✅                  |
| Kombiniert mehrere Abschnitte          | ❌                | ❌                    | ✅                  |
| Einsatz bei komplexen Prüfungen        | ❌                | ❌                    | ✅                  |
| Typisches Beispiel                     | einfache Prüfung  | JOIN-View-Updates     | MAX-/AVG-Prüfungen  |

---

## 📦 Empfehlung – Wann welchen Trigger verwenden?

| Situation                                            | Verwenden               |
|-----------------------------------------------------|--------------------------|
| Einfache Prüfung pro Zeile                          | 🟡 Normal Trigger         |
| INSERT oder UPDATE auf einem View mit berechneten Werten | 🔵 INSTEAD OF Trigger     |
| Zugriff auf die Tabelle UND Zeilendaten nötig       | 🧩 COMPOUND Trigger       |
| Komplexe Logik über mehrere Zeilen                  | 🧩 COMPOUND Trigger       |

