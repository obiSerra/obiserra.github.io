---
layout: post
title:  "Async await"
date:   2019-06-22 00:27:22 +0200
categories: javascript async
---

## The problem

Our node program has to perform some operation in a precise order.
In details, we want to:

1. read a json file
2. compile an Handlebar template with the result data
3. copy all the assets in the output directory
4. produce an html and a pdf from the compiled html

The first step is writing some asynchronous function that will return a Promise.
Then we will put everything togheter using both `async`/`await` and the classic then chaining method


## The code

### Read the json file and compiling the template:

Both functions performs asynchronous operations returning a promise: `readData` reads a json file, while `compileTemplate` compile the Handlebar template with the given data.

```javascript

function readData () {
    return new Promise((resolve, reject) => {
        fs.readFile(dataFile, 'utf8', (error, data) => {
            if (error) reject(error);
            resolve(JSON.parse(data));
        });
    });
}


function compileTemplate (data) {
    return new RSVP.Promise((resolve, reject) => {
        fs.readFile(templateDir + templateFile, 'utf-8', function(error, source) {
            if (error) reject(error);
            const  template = handlebars.compile(source);
            resolve(template(data));
        });
    });
}

```

Once we have our compiled template we need to copy all the assets and generate the output.

Is important to notice that we must wait for all the assets to be copied before starting to generate the PDF, while for the html version we don't really care if all the assets has already been copied.

The `copyAsset` function copy a file using node fs streams returning a promise, the `copyAllAssets` maps the previous function over an array of assets and using `Promise.all` return a promise that get resolved once all the file has been copied. `htmlOutput` and `pdfOutput` will generate the output file.

```javascript

function copyAsset (ast) {
    return new Promise((resolve, reject) => {
        const rStream = fs.createReadStream(templateDir + ast);
        const wStream = fs.createWriteStream('output/' + ast);
        
        rStream.on('error', error => reject(error));
        wStream.on('error', error => reject(error));

        wStream.on('close', () => resolve());

        rStream.pipe(wStream);

    });
}

function copyAllAssets (asts) {
    return Promise.all(assets.map(copyAsset));
}
  
function pdfOutput (html) {
    const options = { format: 'A4', base: basePath, timeout: 30000 };
    return new RSVP.Promise((resolve, reject) => {
        pdf.create(html, options).toFile(fileOut, (error, res) => {
            if (error) reject(error);
            resolve(res);
        });
    });
}

const htmlOutput = (html) => {
    return new Promise ((resolve, reject) => {
        const fileName = 'output/index.html';
        const stream = fs.createWriteStream(fileName);

        stream.once('open', () => stream.end(html));
        stream.on('close', () => resolve());
    });   
};

```

### Putting everything together

We need to declare an async function, in order to use `await`.

`await` tells JavaScript to wait for the promise returned to be resolved before moving to the next instruction.

Let's notice how we used `await` for all the asynchronous operation but for the `htmlOutput`: no other operation depends on the html generation.

```javascript

async function run () {
    try {
        
        const data = await readData();
        console.log('[+] Compiling the template');
        const html = await compileTemplate(data);
        htmlOutput(html);
        console.log('[+] Copy assets');         
        await copyAllAssets(assets);
        console.log('[+] Generating PDF');
        const res = await pdfOutput(html);

        console.log(`[+] Done ${res.filename}`);
    } catch (e) {
        console.log(e);
    }
}

```

The classic version:


```javascript

readData()
.then(d => {
    console.log('[+] Compiling the template');
    return compileTemplate(d)
    })
.then(html => { 
    htmlOutput(html);
    
    console.log('[+] Copy assets');         
    return copyAllAssets(assets).then(() => {
    console.log('[+] Generating PDF');
        return pdfOutput(html);
    });
})
.then(() => console.log(`[+] Done ${res.filename}`))
.catch(e => console.log(e))

```

Using `async` and `await` we can easily write asynchronous code in a sync fashion, simplyfing the flow of our software and making it more maintainable.
