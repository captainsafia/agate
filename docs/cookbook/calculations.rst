====================
Computing new values
====================

Annual change
=============

You could use a :class:`.Formula` to calculate percent change, however, for your convenience agate has a built-in shortcut. For example, if your spreadsheet has a column with values for each year you could do:

.. code-block:: python

    new_table = table.compute([
        ('2000_change', Change('2000', '2001'),
        ('2001_change', Change('2001', '2002'),
        ('2002_change', Change('2002', '2003')
    ])

Or, better yet, compute the whole decade using a loop:

.. code-block:: Python

    computations = []

    for year in range(2000, 2010):
        change = Change(year, year + 1)
        computations.append(('%i_change' % year, change))

    new_table = table.compute(computations)

Annual percent change
=====================

Want percent change instead of value change? Just swap out the :class:`.Aggregation`:

.. code-block:: Python

    computations = []

    for year in range(2000, 2010):
        change = PercentChange(year, year + 1)
        computations.append(('%i_change' % year, change))

    new_table = table.compute(computations)

Indexed/cumulative change
=========================

Need your change indexed to a starting year? Just fix the first argument:

.. code-block:: Python

    computations = []

    for year in range(2000, 2010):
        change = Change(2000, year + 1)
        computations.append(('%i_change' % year, change))

    new_table = table.compute(computations)

Of course you can also use :class:`.PercentChange` if you need percents rather than values.

Round to two decimal places
===========================

agate stores numerical values using Python's :class:`decimal.Decimal` type. This data type ensures numerical precision beyond what is supported by the native :func:`float` type, however, because of this we can not use Python's builtin :func:`round` function. Instead we must use :meth:`decimal.Decimal.quantize`.

We can use :meth:`.Table.compute` to apply the quantize to generate a rounded column from an existing one:

.. code-block:: python

    from decimal import Decimal
    from agate import NumberType

    number_type = NumberType()

    def round_price(row):
        return row['price'].quantize(Decimal('0.01'))

    new_table = table.compute([
        ('price_rounded', Formula(number_type, round_price))
    ])

To round to one decimal place you would simply change :code:`0.01` to :code:`0.1`.

Levenshtein edit distance
=========================

The Levenshtein edit distance is a common measure of string similarity. It can be used, for instance, to check for typos between manually-entered names and a version that is known to be spelled correctly.

Implementing Levenshtein requires writing a custom :class:`.Computation`. To save ourselves building the whole thing from scratch, we will lean on the `python-Levenshtein <https://pypi.python.org/pypi/python-Levenshtein/>`_ library for the actual algorithm.

.. code-block:: python

    import agate
    from Levenshtein.StringMatcher import StringMatcher
    import six

    class LevenshteinDistance(agate.Computation):
        """
        Computes Levenshtein edit distance between the column and a given string.
        """
        def __init__(self, column_name, compare_string):
            self._column_name = column_name
            self._matcher = StringMatcher(seq2=six.text_type(compare_string))

        def get_computed_column_type(self, table):
            """
            The return value is a numerical distance.
            """
            return agate.NumberType()

        def prepare(self, table):
            """
            Verify the comparison column is a TextColumn.
            """
            column = table.columns[self._column_name]

            if not isinstance(column, agate.TextColumn):
                raise agate.UnsupportedComputationError(self, column)

        def run(self, row):
            """
            Find the distance, returning null when the input column was null.
            """
            val = row[self._column_name]

            if val is None:
                return None

            self._matcher.set_seq1(val)

            return self._matcher.distance()

This code can now be applied to any :class:`.Table` just as any other :class:`.Computation` would be:

.. code-block:: python

    new_table = table.compute([
        ('distance', LevenshteinDistance('column_name', 'string to compare'))
    ])

The resulting column will contain an integer measuring the edit distance between the value in the column and the comparison string.
