.. 
   ****************************************************************************
    pgRouting Manual
    Copyright(c) pgRouting Contributors

    This documentation is licensed under a Creative Commons Attribution-Share  
    Alike 3.0 License: http://creativecommons.org/licenses/by-sa/3.0/
   ****************************************************************************

.. _trsp:

pgr_trsp - Turn Restriction Shortest Path (TRSP)
===============================================================================

.. index:: 
	single: pgr_trsp(text,integer,integer,boolean,boolean)
	single: pgr_trsp(text,integer,integer,boolean,boolean,text)
	single: pgr_trsp(text,integer,double precision,integer,double precision,boolean,boolean)
	single: pgr_trsp(text,integer,double precision,integer,double precision,boolean,boolean,text)
	module: trsp

Name
-------------------------------------------------------------------------------

``pgr_trsp`` — Returns the shortest path with support for turn restrictions.


Synopsis
-------------------------------------------------------------------------------

The turn restricted shorthest path (TRSP) is a shortest path algorithm that can optionally take into account complicated turn restrictions like those found in real work navigable road networks. Performamnce wise it is nearly as fast as the A* search but has many additional features like it works with edges rather than the nodes of the network. Returns a set of :ref:`pgr_costResult <type_cost_result>` (seq, id1, id2, cost) rows, that make up a path.

.. code-block:: sql

	pgr_costResult[] pgr_trsp(sql text, source integer, target integer,
                     directed boolean, has_rcost boolean [,restrict_sql text]);


.. code-block:: sql

	pgr_costResult[] pgr_trsp(sql text, source_edge integer, source_pos double precision, 
	                          target_edge integer, target_pos double precision, directed boolean,
                              has_rcost boolean [,restrict_sql text]);


Description
-------------------------------------------------------------------------------

The Turn Restricted Shortest Path algorithm (TRSP) is similar to the :ref:`shooting_star` in that you can specify turn restrictions.

The TRSP setup is mostly the same as :ref:`Dijkstra shortest path <pgr_dijkstra>` with the addition of an optional turn restriction table. This makes adding turn restrictions to a road network much easier than trying to add them to Shooting Star where you had to ad the same edges multiple times if it was involved in a restriction.


:sql: a SQL query, which should return a set of rows with the following columns:

	.. code-block:: sql

		SELECT id, source, target, cost, [,reverse_cost] FROM edge_table


	:id: ``int4`` identifier of the edge
	:source: ``int4`` identifier of the source vertex
	:target: ``int4`` identifier of the target vertex
	:cost: ``float8`` value, of the edge traversal cost. A negative cost will prevent the edge from being inserted in the graph.
	:reverse_cost: (optional) the cost for the reverse traversal of the edge. This is only used when the ``directed`` and ``has_rcost`` parameters are ``true`` (see the above remark about negative costs).

:source: ``int4`` **NODE id** of the start point
:target: ``int4`` **NODE id** of the end point
:directed: ``true`` if the graph is directed
:has_rcost: if ``true``, the ``reverse_cost`` column of the SQL generated set of rows will be used for the cost of the traversal of the edge in the opposite direction.

:restrict_sql: (optional) a SQL query, which should return a set of rows with the following columns:

	.. code-block:: sql

		SELECT to_cost, target_id, via_path FROM restrictions

	:to_cost: ``float8`` turn restriction cost
	:target_id: ``int4`` target id
	:via_path: ``text`` commar seperated list of edges in the reverse order of ``rule``

Another variant of TRSP allows to specify **EDGE id** of source and target together with a fraction to interpolate the position:

:source_edge: ``int4`` **EDGE id** of the start edge
:source_pos: ``float8`` fraction of 1 defines the position on the start edge
:target_edge: ``int4`` **EDGE id** of the end edge 
:target_pos: ``float8`` fraction of 1 defines the position on the end edge

Returns set of :ref:`type_cost_result`:

:seq:   row sequence
:id1:   node ID
:id2:   edge ID (``-1`` for the last row)
:cost:  cost to traverse from ``id1`` using ``id2``


.. rubric:: History

* New in version 2.0.0


Examples
-------------------------------------------------------------------------------

* Without turn restrictions

.. code-block:: sql

	SELECT seq, id1 AS node, id2 AS edge, cost 
		FROM pgr_trsp(
			'SELECT id, source, target, cost FROM edge_table',
			7, 12, false, false
		);

	seq | node | edge | cost 
	----+------+------+------
	  0 |    7 |    6 |    1
	  1 |    8 |    7 |    1
	  2 |    5 |    8 |    1
	  3 |    6 |   11 |    1
	  4 |   11 |   13 |    1
	  5 |   12 |   -1 |    0
	(6 rows)


* With turn restrictions
  
Turn restrictions require additional information, which can be stored in a separate table:

.. code-block:: sql

	CREATE TABLE restrictions (
	    rid serial,
	    to_cost double precision,
	    to_edge integer,
	    from_edge integer,
	    via text
	);

	INSERT INTO restrictions VALUES (1,100,7,4,null);
	INSERT INTO restrictions VALUES (2,4,8,3,5);
	INSERT INTO restrictions VALUES (3,100,9,16,null);

Then a query with turn restrictions is created as:

.. code-block:: sql

	SELECT seq, id1 AS node, id2 AS edge, cost 
		FROM pgr_trsp(
			'SELECT id, source, target, cost FROM edge_table',
			7, 12, false, false, 
			'SELECT to_cost, to_edge AS target_id,
                   from_edge || coalesce('','' || via, '''') AS via_path
               FROM restrictions'
		);

	 seq | node | edge | cost 
	-----+------+------+------
	   0 |    7 |    6 |    1
	   1 |    8 |    7 |    1
	   2 |    5 |    8 |    1
	   3 |    6 |   11 |    1
	   4 |   11 |   13 |    1
	   5 |   12 |   -1 |    0
	(6 rows)

The queries use the :ref:`sampledata` network.


See Also
-------------------------------------------------------------------------------

* :ref:`type_cost_result`
