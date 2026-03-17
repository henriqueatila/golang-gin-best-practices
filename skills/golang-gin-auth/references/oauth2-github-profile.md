# OAuth2 — GitHub Profile Fetcher

See also: `oauth2-flows.md`, `oauth2-google-and-security.md`

## FetchGitHubProfile

Fetches the authenticated user's profile from the GitHub API using the OAuth2 access token.

```go
// internal/auth/github_profile.go
package auth

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
)

type GitHubProfile struct {
    ID    int64  `json:"id"`
    Login string `json:"login"`
    Email string `json:"email"`
}

func FetchGitHubProfile(ctx context.Context, accessToken string) (*GitHubProfile, error) {
    req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://api.github.com/user", nil)
    if err != nil {
        return nil, err
    }
    req.Header.Set("Authorization", "Bearer "+accessToken)
    req.Header.Set("Accept", "application/vnd.github+json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("github api returned %d", resp.StatusCode)
    }

    var profile GitHubProfile
    return &profile, json.NewDecoder(resp.Body).Decode(&profile)
}
```

**Note:** GitHub may not return the email field if the user has set it to private. In that case, call `GET /user/emails` with the `user:email` scope to retrieve verified email addresses.
