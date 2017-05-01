# Database Project 2 - Exbuds
## Linda Du - yld2104	Jason Liang - 
## PostgresSQL - yld2104 

### Expansion:
A private messaging component was implemented. These tables allow for there to be private messages between two or more people. This feature will allow for further interaction between users and option for more private messaging rather than public posts/comments. This further enhances our application by allowing a more variety of interactions between users and thus achieving our goal to encourage exercise.

Added two tables:
``` SQL
Private_Messages
(
	id SERIAL,
	conversationid integer NOT NULL,
	message text,
	sender varchar(15),
	timestamp TIMESTAMP DEFAULT now(),
	FOREIGN KEY (sender) REFERENCES Users ON DELETE CASCADE,
	FOREIGN KEY (conversationid) REFERENCES Conversations,
	PRIMARY KEY(id)
)

Conversations
(
	id SERIAL,
	name VARCHAR(30),
	members VARCHAR(15)[],
	PRIMARY KEY (id)
)
```

#### **TEXT:** 
We model the attribute message as text because messages are more natural language and this will allow for an implementation of extensive search on the messages if desired.

For example, the following would search through all messages that user has access to, in other words messages from conversations they are in, (in this case, Linda's) and return all messages that contain the lexemes provided ('league' in this case):

SELECT pm.id, c.id, message, sender FROM Private_Messages pm, Conversations c WHERE pm.conversationid = c.id AND members @> ARRAY['LindaDu']::varchar(15)[] AND to_tsvector(pm.message) @@ to_tsquery('league');

```
 id | id |                                  message                                  | sender  
----+----+---------------------------------------------------------------------------+---------
  8 |  6 | Anyone up for league tonight? I can play around 12:00 after finishing hw. | LindaDu
(1 row)
```

#### **ARRAY:** 
We keep track of members in a conversation as an array of the usernames. Since Postgres 9.3 does not seem to support Foreign Key attribute on elements of an array, we also added a Trigger that checks before inserts and updates to make sure that elements in the array members are all valid userids.

For example, the following would check if given user (in this case 'LindaDu') is part of a specific conversation (in this case conversation 1) ~ returns 0 if no and 1 if yes:

``` SQL
SELECT COUNT(*) FROM Conversations WHERE id = '1' AND members @> ARRAY['LindaDu']::varchar(15)[];
```

```
 count 
-------
     1
(1 row)
```

#### **TRIGGERS:** 
We added two triggers.
1. As described above, is to implement foreign key contraint on elements of members array in Conversations table. This Trigger is activated before any INSERT or UPDATE on the Conversations table. When trigered, it loops through all elements of the members array and checks if it is a valid username by checking if a query conditional on the username returns any tuples. It raises and exception when there is an invalid userid and doesn't allow the insert/update.

*Example situation:*
Users Table - (just username column)
```
username   
-------------
 JasonLiang
 LinShi
 JaeLee
 AdamCannon
 TOPChoi
 YunaLin
 MichelleLee
 LindaDu
 PeterZheng
 KemingZhang
 ShawnXia
 DavidZhang
 JayChow
 MayLi
 SherryZhu
 kshi
 GDragon
 PaulBlaer
 lilywu
 jason808
 Tondieker
 nami
 lalaland
 test
(24 rows)
```
``` SQL
INSERT INTO Conversations (name,members) VALUES ('Team','{lulu,LindaDu,kshi,JaeLee}');
```
> ERROR:  user lulu does not exist

*~ Code ~*
``` SQL
CREATE FUNCTION checkConversation() RETURNS trigger AS $conversation_stamp$
	DECLARE	
		myrec record;
		x varchar(15);
	BEGIN
		FOREACH x IN ARRAY NEW.members
		LOOP
			SELECT * INTO myrec FROM Users WHERE username = x;
			IF NOT FOUND THEN 
				RAISE EXCEPTION 'user % does not exist', x;
			END IF;
		END LOOP;

		RETURN NEW;
	END;
$conversation_stamp$ LANGUAGE plpgsql;

CREATE TRIGGER conversation_mem
BEFORE UPDATE OR INSERT ON Conversations
	FOR EACH ROW EXECUTE PROCEDURE checkConversation();
```

2. Added a trigger to make sure that a user can only send a private message in a conversation that they are a member of. Since these conversations are private, we must insure that users only have access to send messages in conversations they are a member of. Since this requirement involves both tables, we cannot simply model this with a constraint. Thus, a trigger function seems appropriate to handle this. The trigger is activated before any INSERT or UPDATE on the table Private_Messages. Once activated, it checks if any tuples are returned by a query on the Conversations table conditional on given conversation id (id = NEW.conversationid) AND that sender is contained in the members array of the matching conversation (members @> ARRAY[NEW.sender]::varchar(15)[]). If there is no tuple, then it raises an exception for invalid sender ('User % not part of conversation %').

*Example situaton:*
Conversations table - 
```
id |   name    |             members             
----+-----------+---------------------------------
  1 | Team Zeta | {LindaDu,kshi,JaeLee}
  2 | Tues Ball | {JasonLiang,kshi,JaeLee}
  3 |           | {MichelleLee,Tondieker}
  4 | Crypto    | {AdamCannon,JaeLee,JasonLiang}
  5 | KPop      | {TOPChoi,YunaLin,GDragon,MayLi}
  6 | Leaguers  | {JasonLiang,kshi,LindaDu,nami}
  7 | Music     | {JayChow,GDragon,JaeLee}
  8 |           | {LindaDu,kshi}
  9 |           | {JasonLiang,Tondieker}
 10 |           | {JayChow,JaeLee}
(10 rows)
```

``` SQL
INSERT INTO Private_Messages(conversationid,message,sender) VALUES ('3','helllooo, this is a test', 'LindaDu');
```
> ERROR:  User LindaDu not part of conversation 3

*~ Code ~*
``` SQL
CREATE FUNCTION checkMessages() RETURNS trigger AS $message_stamp$
	DECLARE
		mess_rec record;
	BEGIN
		SELECT * INTO mess_rec FROM Conversations WHERE id = NEW.conversationid AND members @> ARRAY[NEW.sender]::varchar(15)[];
		IF NOT FOUND THEN 
			RAISE EXCEPTION 'User % not part of conversation %', NEW.sender, NEW.conversationid;
		END IF;

		RETURN NEW;
	END;
$message_stamp$ LANGUAGE plpgsql;

CREATE TRIGGER message_update
	BEFORE UPDATE OR INSERT ON Private_Messages
		FOR EACH ROW EXECUTE PROCEDURE checkMessages();
```
