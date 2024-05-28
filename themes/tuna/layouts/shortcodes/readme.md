{{ $repo := .Get 0 }}
{{ $json := getJSON "https://api.github.com/repos/mulling/" $repo "/readme" }}
{{ $json.content | base64Decode | markdownify }}
