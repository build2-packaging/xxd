# xxd - Vim hexdump utility

This is a `build2` package for the Vim
[`xxd`](https://github.com/vim/vim/tree/master/src/xxd) executable. It creates
a hexdump of a given file or standard input. It can also convert a hexdump
back to its original binary form. One notable `xxd` mode is `-i|--include`,
which generates a C array definition of the input that can be used to embed
binary data into C/C++ programs. This functionality is the primary reason for
the use of this package.

Note that the `xxd` executable in this page provides `build2` metadata.


## Usage

To start using `xxd` in your project, add the following build-time `depends`
value to your `manifest`, adjusting the version constraint as appropriate:

```
depends: * xxd >= 8.2.3075
```

Then import the executable in your `buildfile`:

```
import! [metadata] xxd = xxd%exe{xxd}
```

While the default output of the `-i|--include` mode is a bit old school (using
`unsigned int` instead of `size_t`) and the array/length names are derived
from the input file name (including directories), `xxd` can also produce just
the array values allowing us to wrap it into an array of our choice. Below are
some examples of `build2` ad hoc recipes:

```
# Use the default output. Array and length names will be binary_bin and
# binary_bin_len.
#
cxx{binary}: file{binary.bin} $xxd
{{
  i = $path($<[0])
  env --cwd $directory($i) -- $xxd -i $leaf($i) >$path($>)
}}
```

```
# Use just the file name without the extension for names and size_t for
# length.
#
cxx{binary}: file{binary.bin} $xxd
{{
  i = $path($<[0]) # Input.
  o = $path($>)    # Output.
  n = $name($<[0]) # Array name.

  echo "#include <cstddef>"                     >$o
  echo "unsigned char $n[] = {"                >>$o
  $xxd -i <$i                                  >>$o
  echo "};"                                    >>$o
  echo "std::size_t $(n)_len = sizeof \($n\);" >>$o
}}
```

```
# Generate both the header with declarations and the source file with
# definitions.
#
<{hxx cxx}{binary}>: file{binary.bin} $xxd
{{
  i = $path($<[0]) # Input.
  h = $path($>[0]) # Output header.
  s = $path($>[1]) # Output source.
  n = $name($<[0]) # Array name.

  # Get the position of the last byte (in hex).
  #
  $xxd -s -1 -l 1 $i | sed -n -e 's/^([0-9a-f]+):.*$/\1/p' - | set pos

  if ($empty($pos))
    exit "unable to extract input size from xxd output"
  end

  # Write the header and source.
  #
  echo "#pragma once"                          >$h
  echo "extern unsigned char $n[0x$pos + 1];" >>$h

  echo "unsigned char $n[0x$pos + 1] = {"      >$s
  $xxd -i <$i                                 >>$s
  echo '};'                                   >>$s
}}
```
