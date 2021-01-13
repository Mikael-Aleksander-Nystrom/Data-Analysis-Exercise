--Luodaan teht‰v‰nannon taulut

CREATE TABLE level_events (
	player_id int,
	stamp timestamp,
	level_number int
);

CREATE TABLE installation_events (
	player_id int,
	stamp timestamp
);


--Importataan CSV-tiedostojen sis‰llˆt tauluihinsa


--Joinataan taulut yhteen, jotta p‰‰st‰‰n p‰iv‰m‰‰riin k‰siksi age-sarakkeen laskemiseksi

CREATE TABLE ages AS (
	SELECT level_events.player_id, level_events.level_number, level_events.stamp,
		installation_events.stamp as installation
	FROM level_events
	JOIN installation_events
	ON level_events.player_id = installation_events.player_id
	);

--Lis‰t‰‰n age-sarake ja lasketaan pelaajien ik‰ kullakin levelill‰

ALTER TABLE ages
ADD COLUMN age int;
UPDATE ages SET age = DATE_PART('day',stamp-installation);

--Poistetaan turhat sarakkeet

ALTER TABLE ages
DROP COLUMN stamp,
DROP COLUMN installation;

--Luodaan taulu, jossa sarake progress kuvaa teht‰v‰nannon mukaista progress-lukua

CREATE TABLE progress AS(
SELECT player_id, age, max(level_number) as progress
FROM ages
GROUP BY player_id, age
ORDER BY player_id, age
);


--Luodaan taulu, johon importataan manuaalisesti age-luvut 0-177.
--Tarvitsemamme age-intervalli voidaan tutkia max()-funktiota k‰ytt‰en
--T‰m‰n voisi varmaan tehd‰ k‰tev‰mminkin, niin v‰ltytt‰isiin kovakoodaamiselta

CREATE TABLE tyhja (
	age int
);


--Luodaan laajennettu taulu, jossa jokaiselle pelaaja-id:lle on oma rivi kutakin age-arvoa kohti.

CREATE TABLE laajennettu AS(
SELECT player_id, tyhja.age
FROM installation_events
CROSS JOIN tyhja
);

--Luodaan taulu, jossa kunkin pelaajan progress n‰kyy kullakin age-luvulla

CREATE TABLE overall_progress AS (
SELECT laajennettu.player_id, laajennettu.age, progress.progress
FROM laajennettu
LEFT JOIN progress
ON laajennettu.player_id = progress.player_id AND laajennettu.age = progress.age
ORDER BY player_id, age
);


--Asetetaan alkuprogress nollaksi niille pelaajille, jotka eiv‰t suorita tasoja asennusp‰iv‰n‰

UPDATE overall_progress
SET progress = 0
WHERE progress IS NULL AND age = 0;


--T‰ytet‰‰n progress-sarakkeen NULL-arvot kunkin pelaajan silloisella levelill‰

CREATE TABLE total_progress AS(
	WITH C AS
(
  SELECT player_id, age, progress,
    MAX( CASE WHEN progress IS NOT NULL THEN player_id END )
  OVER( ORDER BY player_id, age
        ROWS UNBOUNDED PRECEDING ) AS grp
  FROM overall_progress
)
SELECT player_id, age,
  MAX(progress) OVER( PARTITION BY grp
          ORDER BY player_id, age
          ROWS UNBOUNDED PRECEDING ) AS progress
FROM C
	);


--Total averaget
CREATE TABLE avg_all AS (
SELECT age, avg(progress)
FROM total_progress
GROUP BY age
ORDER BY age
);

--Active averaget
CREATE TABLE avg_active AS (
SELECT age, avg(progress)
FROM progress
GROUP BY age
ORDER BY age
);

--Luodaan viimeinen, kysytynlainen output-taulu

DROP TABLE IF EXISTS averages;
CREATE TABLE averages AS (
SELECT avg_all.age, avg_active.avg as average_level_active, avg_all.avg as average_level_all
FROM avg_all
LEFT JOIN avg_active
ON avg_all.age = avg_active.age
);

--Ja lopuksi exportataan taulu averages tiedostoksi averages.cvs, jotta siihen p‰‰st‰‰n k‰tev‰sti k‰siksi Pythonilla.