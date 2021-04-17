## Apache2 / httpd

### What is Apache?

Apache is a program that will essentially host a file directory. It keeps a port open on 80 that allows other computers to request files. Apache will look for those files and then send the file's contents to whoever requested them.

### Customizing Apache

You can for example use .htaccess files to modify how Apache behaves. For example, using RewriteRule, you can use regex to tell Apache how to turn URLs into filepaths beyond the way it does it by default. For example, by choosing to interpret the url `myblog.com/aboutme` as the filepath `aboutme.php`, as opposed to the default `aboutme`. We used RewriteRule to tell it about the `.php` file ending. Additionally, Apache can be told to first execute files that end in `.php` using the program `php`, and then returning the result - as opposed to simply returning the original php file. The same can be done with other programming langauges like Python.
