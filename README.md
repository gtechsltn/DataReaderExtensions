# DataReaderExtensions
* System.Data.**IDataReader**
* System.Data.**IDataRecord**
* System.Collections.**IEnumerable**
* C# **yield return**
* C# **dynamic**
* C# **byte[]**
* SQL Server **varbinary(MAX)**
* https://github.com/gtechsltn/ADO-DataReaderExtensions
* https://github.com/gtechsltn/DataReaderExtensions
* [Extension methods for the DataReaderExtensions class](https://jonlabelle.com/snippets/raw/677/DataReaderExtensions.cs)
* [DataReaderExtensions](https://fluentmigrator.github.io/api/v3.x/FluentMigrator.Runner.Processors.DataReaderExtensions.html)
* [System.Data.IDataReader extension method to fully read the bytes from a large binary column](https://gist.github.com/gtechsltn/fe436f6e637fd1c6de1675a97bf045ae)
* [DataReaderExtensions](https://gist.github.com/gtechsltn/3bf15e938fc8e6e4b910dc00306d6332)
* [Some handy utilities for dealing with ado.net](https://gist.github.com/gtechsltn/2204b09420723ef5e5e56bb1d0b9755c)

## DataReaderExtensions Interface
```
public static DataSet ReadDataSet(this IDataReader reader)
{
}
public static DataTable ReadTable(this IDataReader reader)
{
}
public static byte[] GetBytes(this IDataReader reader, int column)
{
}
public static dynamic ToRow(this IDataRecord record)
{
}
```

## DataReaderExtensions.cs
```
using System.Collections.Generic;
using System.Data;

// A set of extension methods for IDataReader
public static class DataReaderExtensions
{
	// Enumerates through the reads in an IDataReader.
	public static IEnumerable<IDataRecord> AsEnumerable(this IDataReader reader)
	{
		while (reader.Read())
		{
			yield return reader;
		}
	}
}
```

## Usage of DataReaderExtensions.cs
```
using (var connection = new SqlConnection("some connection string"))
{
    using (var command = new SqlCommand("select * from products", connection))
    {
        connection.Open();

        using (var reader = command.ExecuteReader())
        {
            var results = reader.AsEnumerable()
                .Select(record => new Product
                            {
                                Name = (string)record["product_name"],
                                Id = (int)record["product_id"],
                                Category = (string)record["product_category"]
                            })
                .GroupBy(product => product.Category);
        }
    }
}
```

## DataReaderExtensions.cs
```
using System.Data;
using System.IO;

namespace Extensions.Data {

    public static class DataReaderExtensions {

        /*
         * Extension method on IDataReader that reads the entire contents of a large binary column using a single method call.
         * Sample usage:
         *   byte[] bytes = reader.GetBytes(columnIndex);
         */
        public static byte[] GetBytes(this IDataReader reader, int column) {
            using (MemoryStream ms = new MemoryStream()) {
                byte[] buff = new byte[8192];
                long offset = 0L;
                long n = 0L;
                do {
                    n = reader.GetBytes(column, offset, buff, 0, buff.Length);
                    ms.Write(buff, 0, (int)n);
                    offset += n;
                } while (n >= buff.Length);
                return ms.ToArray();
            }
        }
    }
}
```

If you feel that the IDataReader interface could use some work, here is your change. DataReaderExtensions focuses on empowering the IDataReader interface through new method extensions that will make your code a lot cleaner.

# Simple Type Getters

This is a very simple library. Here is the summary of methods that it adds to the IDataReader interface.

Let's assume we have the following variable

```#
IDataReader reader;
```

We have methods for all types:

```C#
reader.GetString
reader.GetBoolean
reader.GetByte
reader.GetDecimal
reader.GetDouble
reader.GetFloat
reader.GetInt16
reader.GetInt32
reader.GetInt64
reader.GetDateTime
```

So let's suppose we have a column named "MyTableID" which is an integer, we can do:

```C#
var id = reader.GetInt32("MyTableID");
```

You might be thinking this is trivial, why do we have a library for this? Well, you would have to do this with an IDataReader.

```C#
var ordinal = reader.GetOrdinal("MyTableID");
var id = reader.GetInt32(ordinal);
```

Sure, you might put it in one line, but that's not clean.

# Nullable Type Getters

Now we have a more interesting case. We might have a value that might be NULLABLE, but on top of that, IDataReader returns DBNull instead of null directly. 

This is what your code would look like, assuming your field is a DateTime. I am not using var so it is clear which types we are handling.

```C#
int ordinal = reader.GetOrdinal("NullableDate");
if (reader.IsDBNull(ordinal))
    return;

DateTime value = reader.GetDateTime(ordinal);
```

I tried to be clean. But man that's a bunch of lines for such a simple operation! Let's see what happens when we use our library.

```C#
var value = reader.GetDateTimeNullable("NullableDate");
```

You're welcome.

The full list of Nullable methods:

```C#
reader.GetString
reader.GetBooleanNullable
reader.GetByteNullable
reader.GetDecimalNullable
reader.GetDoubleNullable
reader.GetFloatNullable
reader.GetInt16Nullable
reader.GetInt32Nullable
reader.GetInt64Nullable
reader.GetDateTimeNullable
```

reader.GetString is nullable by default, the library will handle this case for you always.

# Byte Array Getter

This is a special type of getter. If you use VARBINARY fields, or anything that returns an array of bytes, this method is for you!

Let's see what we would need to do to read a column named "BinaryData" which is of type VARBINARY(MAX) (typical scenario).

Assume a variable reader of type IDataReader.

We are not using var so the types are clear in this example.

```C#
int ordinal = reader.GetOrdinal("BinaryData");
if (reader.IsDBNull(ordinal))
    return default(byte[]);

long dataLength = reader.GetBytes(ordinal, 0, null, 0, 0);
byte[] bytes = new byte[dataLength];
int bufferSize = 1024;
long bytesRead = 0L;
int curPos = 0;
while (bytesRead < dataLength)
{
    bytesRead += reader.GetBytes(ordinal, curPos, bytes, curPos, bufferSize);
    curPos += bufferSize;
}

return bytes;
```

That's a bunch of work! Seems like .NET's System.Data is punishing us for saving binary data into a database. Well, thankfully our library fixes this problem.

```C#
return reader.GetBytes("BinaryData");
```

Again, you are welcome!

If you find cases where you need to return an array of bytes, but this method does not work, then please send me the column definition and what you are trying to do as an Issue in GitHub. I would love to create a solution for you.

## DataReaderExtensions.cs
```
using System;
using System.ComponentModel;
using System.Data;

namespace Extensions
{
    public static class DataReaderExtensions
    {
        public static string GetSafeString(this IDataReader reader, string columnName)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetString(index) : string.Empty;
        }

        public static string GetSafeString(this IDataReader reader, string columnName, string defaultValue)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetString(index) : defaultValue;
        }

        public static int? GetSafeInt(this IDataReader reader, string columnName)
        {
            return reader.GetSafeInt32(columnName);
        }

        public static int GetSafeInt(this IDataReader reader, string columnName, int defaultValue)
        {
            return reader.GetSafeInt32(columnName, defaultValue);
        }

        public static int? GetSafeInt32(this IDataReader reader, string columnName)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? (int?)reader.GetInt32(index) : null;
        }

        public static int GetSafeInt32(this IDataReader reader, string columnName, int defaultValue)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetInt32(index) : defaultValue;
        }

        public static long? GetSafeInt64(this IDataReader reader, string columnName)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? (long?)reader.GetInt64(index) : null;
        }

        public static long GetSafeInt64(this IDataReader reader, string columnName, long defaultValue)
        {
            // DB type: bigint
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetInt64(index) : defaultValue;
        }

        public static short? GetSafeInt16(this IDataReader reader, string columnName)
        {
            // DB type: smallint
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? (short?)reader.GetInt16(index) : null;
        }

        public static short GetSafeInt16(this IDataReader reader, string columnName, short defaultValue)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetInt16(index) : defaultValue;
        }

        public static Guid GetSafeGuid(this IDataReader reader, string columnName, Guid defaultValue)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetGuid(index) : defaultValue;
        }

        public static Guid? GetSafeGuid(this IDataReader reader, string columnName)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? (Guid?)reader.GetGuid(index) : null;
        }

        public static Guid GetSafeGuidOrEmpty(this IDataReader reader, string columnName)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetGuid(index) : Guid.Empty;
        }

        public static DateTime? GetSafeDateTime(this IDataReader reader, string columnName)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? (DateTime?)reader.GetDateTime(index) : null;
        }

        public static DateTime GetSafeDateTime(this IDataReader reader, string columnName, DateTime defaultValue)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetDateTime(index) : defaultValue;
        }

        public static dynamic GetSafeDateTime(this IDataReader reader, string columnName, string defaultValue)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? (dynamic)reader.GetDateTime(index) : defaultValue;
        }

        public static DateTimeOffset GetSafeDateTimeOffset(this IDataReader reader, string columnName, DateTimeOffset defaultValue)
        {
            // Gets the record value casted as DateTimeOffset (UTC) or the specified default value.
            int index = reader.GetOrdinal(columnName);
            if (!reader.IsDBNull(index))
            {
                DateTime dt = reader.GetDateTime(index);
                if (dt != DateTime.MinValue)
                {
                    return new DateTimeOffset(dt, TimeSpan.Zero);
                }
            }

            return defaultValue;
        }

        public static bool? GetSafeBool(this IDataReader reader, string columnName)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? (bool?)reader.GetBoolean(index) : null;
        }

        public static bool GetSafeBool(this IDataReader reader, string columnName, bool defaultValue)
        {
            int index = reader.GetOrdinal(columnName);

            return !reader.IsDBNull(index) ? reader.GetBoolean(index) : defaultValue;
        }

        public static T Fill<T>(this IDataReader reader, T obj)
        {
            foreach (PropertyDescriptor prop in TypeDescriptor.GetProperties(obj))
            {
                if (!prop.IsReadOnly)
                {
                    if (reader.FieldExists(prop.Name))
                    {
                        if (reader[prop.Name] is DBNull)
                        {
                            prop.SetValue(obj, null);
                        }
                        else
                        {
                            prop.SetValue(obj, reader[prop.Name]);
                        }
                    }
                }
            }

            return obj;
        }

        public static bool FieldExists(this IDataRecord record, string columnName)
        {
            for (int i = 0; i < record.FieldCount; i++)
            {
                if (record.GetName(i).Equals(columnName, StringComparison.InvariantCultureIgnoreCase))
                {
                    return true;
                }
            }

            return false;
        }

        /// <summary>
        /// Returns the index of a column by name or -1.
        /// </summary>
        /// <param name="this">The data record.</param>
        /// <param name="name">The field name (case insensitive).</param>
        /// <returns>The index of a column by name, or -1.</returns>
        public static int IndexOf(this IDataRecord record, string name)
        {
            for (int i = 0; i < record.FieldCount; i++)
            {
                if (String.Compare(record.GetName(i), name, StringComparison.InvariantCultureIgnoreCase) == 0)
                {
                    return i;
                }
            }

            return -1;
        }

        public static bool IsDbNull(this IDataReader reader, string field)
        {
            return reader[field] == DBNull.Value;
        }

        /// <summary>
        /// Reads all all records from a data reader and performs an
        /// action for each.
        /// </summary>
        /// <param name="reader">The data reader.</param>
        /// <param name="action">The action to be performed.</param>
        /// <returns>The count of actions that were performed.</returns>
        public static int ReadAll(this IDataReader reader, Action<IDataReader> action)
        {
            int count = 0;

            while (reader.Read())
            {
                action(reader);
                count++;
            }

            return count;
        }
    }
}
```

## DataReaderExtension.cs
```
public static class DataReaderExtension
{
    public static IEnumerable<Object[]> DataRecord(this System.Data.IDataReader source)
    {
        if (source == null)
            throw new ArgumentNullException("source");
 
        while (source.Read())
        {
            Object[] row = new Object[source.FieldCount];
            source.GetValues(row);
            yield return row;
        }
    }
}
```
