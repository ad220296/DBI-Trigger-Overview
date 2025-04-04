
# 🔄 Vergleich & Beispiele: Trigger in PL/SQL (DBI-Doku)

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

## 🧩 3. COMPOUND Trigger (Tabelle)

Ein **Compound Trigger** kombiniert mehrere Triggerbereiche:

- `BEFORE STATEMENT`
- `BEFORE EACH ROW`
- `AFTER EACH ROW`
- `AFTER STATEMENT`

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

