FROM httpd:2.4
# Activar módulos LDAP necesarios
RUN sed -i \
    -e 's/^#\(LoadModule authnz_ldap_module modules\/mod_authnz_ldap.so\)/\1/' \
    -e 's/^#\(LoadModule ldap_module modules\/mod_ldap.so\)/\1/' \
    -e 's/^#\(LoadModule rewrite_module modules\/mod_rewrite.so\)/\1/' \
    conf/httpd.conf

# Permitir .htaccess (AllowOverride All)
RUN sed -i '/<Directory "\/usr\/local\/apache2\/htdocs">/,/<\/Directory>/ s/AllowOverride None/AllowOverride All/' conf/httpd.conf