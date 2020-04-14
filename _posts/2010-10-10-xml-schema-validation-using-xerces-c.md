---
layout: post
title: "XML Schema Validation using Xerces-C++"
date: 2010-10-10 22:22:00 +0530
tags: [c-cpp, xml]
---
Recently I've been writing some serious implementation in Xerces-C++, Xml-Security-C++ and OpenSSL. I planned to share my experience in a series of posts. In this post I am going to show how can we validate a XML against its schema using Xerces-C++. To demonstrate the schema validation I am going to use the example schema and XML from W3 Schools. Our schema file is shiporder.xsd and the XML file is shiporder.xml. Lets jump into the code

```c++
#include <stdio.h>

#include <xercesc/util/XMLString.hpp>
#include <xercesc/parsers/XercesDOMParser.hpp>
#include <xercesc/framework/LocalFileInputSource.hpp>
#include <xercesc/sax/ErrorHandler.hpp>
#include <xercesc/sax/SAXParseException.hpp>
#include <xercesc/validators/common/Grammar.hpp>

XERCES_CPP_NAMESPACE_USE

class WStr
{
    private:
        XMLCh*  wStr;

    public:
        WStr(const char* str)
        {
            wStr = XMLString::transcode(str);
        }

        ~WStr()
        {
            XMLString::release(&amp;wStr);
        }

        operator const XMLCh*() const
        {
            return wStr;
        }
};

class ParserErrorHandler : public ErrorHandler
{
    private:
        void reportParseException(const SAXParseException&amp; ex)
        {
            char* msg = XMLString::transcode(ex.getMessage());
            fprintf(stderr, "at line %llu column %llu, %s\n", 
                    ex.getLineNumber(), ex.getColumnNumber(), msg);
            XMLString::release(&amp;msg);
        }

    public:
        void warning(const SAXParseException&amp; ex)
        {
            reportParseException(ex);
        }

        void error(const SAXParseException&amp; ex)
        {
            reportParseException(ex);
        }

        void fatalError(const SAXParseException&amp; ex)
        {
            reportParseException(ex);
        }

        void resetErrors()
        {
        }
};

void ValidateSchema(const char* schemaFilePath, const char* xmlFilePath)
{
    XercesDOMParser domParser;
    if (domParser.loadGrammar(schemaFilePath, Grammar::SchemaGrammarType) == NULL)
    {
        fprintf(stderr, "couldn't load schema\n");
        return;
    }

    ParserErrorHandler parserErrorHandler;

    domParser.setErrorHandler(&amp;parserErrorHandler);
    domParser.setValidationScheme(XercesDOMParser::Val_Auto);
    domParser.setDoNamespaces(true);
    domParser.setDoSchema(true);
    domParser.setValidationConstraintFatal(true);

    domParser.parse(xmlFilePath);
    if (domParser.getErrorCount() == 0)
        printf("XML file validated against the schema successfully\n");
    else
        printf("XML file doesn't conform to the schema\n");
}

int main(int argc, const char *argv[])
{
    if (argc != 3)
    {
        printf("SchemaValidator <schema file> <xml file>\n");
        return 0;
    }

    XMLPlatformUtils::Initialize();

    ValidateSchema(argv[1], argv[2]);

    XMLPlatformUtils::Terminate();

    return 0;
}
```

The classes WStr and ParserErrorHandler are helper classes. The class WStr helps us to convert UTF-8 string to UTC-16 string with auto pointer like usage. The class ParserErrorHandler helps to show the parser errors to the console. The function ValidateSchema does the actual validation. The important lines are
```c++
domParser.setValidationScheme(XercesDOMParser::Val_Auto);
domParser.setDoNamespaces(true);
domParser.setDoSchema(true);
domParser.setValidationConstraintFatal(true);
```
which enables namespace processing and schema validation. It also tells the parser to consider any validation error as fatal. The command line to compile the SchemaValidator code is
```bash
$ g++ SchemaValidator.cpp -o SchemaValidator -lxerces-c
```
Lets try our SchemaValidator with valid and invalid xml. First lets try with valid xml file
```bash
$ ./SchemaValidator shiporder.xsd shiporder.xml
```
XML file validated against the schema successfully
and with a invalid shiporder in which I've added a element `oldprice` to the second `item`
```bash
$ ./SchemaValidator shiporder.xsd shiporder-invalid.xml
at line 15 column 23, no declaration found for element 'oldprice'
XML file doesn't conform to the schema
```
You can see the schema validation fails saying `oldprice` is unknown element according to given schema(xsd). I hope I showed you how to validate XML schema using Xerces-C++. We'll see what else we can do with Xerces-C++ in the future posts.