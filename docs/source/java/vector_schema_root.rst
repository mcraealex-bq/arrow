.. Licensed to the Apache Software Foundation (ASF) under one
.. or more contributor license agreements.  See the NOTICE file
.. distributed with this work for additional information
.. regarding copyright ownership.  The ASF licenses this file
.. to you under the Apache License, Version 2.0 (the
.. "License"); you may not use this file except in compliance
.. with the License.  You may obtain a copy of the License at

..   http://www.apache.org/licenses/LICENSE-2.0

.. Unless required by applicable law or agreed to in writing,
.. software distributed under the License is distributed on an
.. "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
.. KIND, either express or implied.  See the License for the
.. specific language governing permissions and limitations
.. under the License.

.. default-domain:: java
.. highlight:: java

============
Tabular Data
============

While arrays (aka: :doc:`ValueVector <./vector>`) represent a one-dimensional sequence of
homogeneous values, data often comes in the form of two-dimensional sets of
heterogeneous data (such as database tables, CSV files...). Arrow provides
several abstractions to handle such data conveniently and efficiently.

Fields
======

Fields are used to denote the particular columns of tabular data.
A field, i.e. an instance of `Field`_, holds together a field name, a data
type, and some optional key-value metadata.

.. code-block:: Java

    // Create a column "document" of string type with metadata
    import org.apache.arrow.vector.types.pojo.ArrowType;
    import org.apache.arrow.vector.types.pojo.Field;
    import org.apache.arrow.vector.types.pojo.FieldType;

    Map<String, String> metadata = new HashMap<>();
    metadata.put("A", "Id card");
    metadata.put("B", "Passport");
    metadata.put("C", "Visa");
    Field document = new Field("document", new FieldType(true, new ArrowType.Utf8(), /*dictionary*/ null, metadata), /*children*/ null);

Schemas
=======

A `Schema`_ describes the overall structure consisting of any number of columns. It holds a sequence of fields together
with some optional schema-wide metadata (in addition to per-field metadata).

.. code-block:: Java

    // Create a schema describing datasets with two columns:
    // a int32 column "A" and a utf8-encoded string column "B"
    import org.apache.arrow.vector.types.pojo.ArrowType;
    import org.apache.arrow.vector.types.pojo.Field;
    import org.apache.arrow.vector.types.pojo.FieldType;
    import org.apache.arrow.vector.types.pojo.Schema;
    import static java.util.Arrays.asList;

    Map<String, String> metadata = new HashMap<>();
    metadata.put("K1", "V1");
    metadata.put("K2", "V2");
    Field a = new Field("A", FieldType.nullable(new ArrowType.Int(32, true)), null);
    Field b = new Field("B", FieldType.nullable(new ArrowType.Utf8()), null);
    Schema schema = new Schema(asList(a, b), metadata);

VectorSchemaRoot
================

.. note::

    VectorSchemaRoot is somewhat analogous to tables and record batches in the other Arrow implementations
    in that they all are 2D datasets, but the usage is different.

A :class:`VectorSchemaRoot` is a container that can hold batches, batches flow through :class:`VectorSchemaRoot`
as part of a pipeline. Note this is different from other implementations (i.e. in C++ and Python,
a :class:`RecordBatch` is a collection of equal-length vector instances and was created each time for a new batch).

The recommended usage for :class:`VectorSchemaRoot` is creating a single :class:`VectorSchemaRoot`
based on the known schema and populated data over and over into the same VectorSchemaRoot in a stream
of batches rather than creating a new :class:`VectorSchemaRoot` instance each time
(see `Flight`_ or ``ArrowFileWriter`` for better understanding). Thus at any one point a VectorSchemaRoot may have data or
may have no data (say it was transferred downstream or not yet populated).

Here is the example of building a :class:`VectorSchemaRoot`

.. code-block:: Java

    BitVector bitVector = new BitVector("boolean", allocator);
    VarCharVector varCharVector = new VarCharVector("varchar", allocator);
    bitVector.allocateNew();
    varCharVector.allocateNew();
    for (int i = 0; i < 10; i++) {
      bitVector.setSafe(i, i % 2 == 0 ? 0 : 1);
      varCharVector.setSafe(i, ("test" + i).getBytes(StandardCharsets.UTF_8));
    }
    bitVector.setValueCount(10);
    varCharVector.setValueCount(10);

    List<Field> fields = Arrays.asList(bitVector.getField(), varCharVector.getField());
    List<FieldVector> vectors = Arrays.asList(bitVector, varCharVector);
    VectorSchemaRoot vectorSchemaRoot = new VectorSchemaRoot(fields, vectors);

The vectors within a :class:`VectorSchemaRoot` could be loaded/unloaded via :class:`VectorLoader` and :class:`VectorUnloader`.
:class:`VectorLoader` and :class:`VectorUnloader` handles converting between :class:`VectorSchemaRoot` and :class:`ArrowRecordBatch` (
representation of a RecordBatch :doc:`IPC <../format/IPC.rst>` message). Examples as below

.. code-block:: Java

    // create a VectorSchemaRoot root1 and convert its data into recordBatch
    VectorSchemaRoot root1 = new VectorSchemaRoot(fields, vectors);
    VectorUnloader unloader = new VectorUnloader(root1);
    ArrowRecordBatch recordBatch = unloader.getRecordBatch();

    // create a VectorSchemaRoot root2 and load the recordBatch
    VectorSchemaRoot root2 = VectorSchemaRoot.create(root1.getSchema(), allocator);
    VectorLoader loader = new VectorLoader(root2);
    loader.load(recordBatch);

A new :class:`VectorSchemaRoot` could be sliced from an existing instance with zero-copy

.. code-block:: Java

    // 0 indicates start index (inclusive) and 5 indicated length (exclusive).
    VectorSchemaRoot newRoot = vectorSchemaRoot.slice(0, 5);

.. _`Field`: https://arrow.apache.org/docs/java/reference/org/apache/arrow/vector/types/pojo/Field.html
.. _`Schema`: https://arrow.apache.org/docs/java/reference/org/apache/arrow/vector/types/pojo/Schema.html
.. _`Flight`: https://arrow.apache.org/docs/java/reference/org/apache/arrow/flight/package-summary.html
